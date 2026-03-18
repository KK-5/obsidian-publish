Atom中bindless实现的方案大致思想：
每一个resource view（buffer view 或 image view等）初始化时会在一个静态的描述符堆中创建描述符，在其生命周期中此描述符的位置不会发生变化。
使用bindless绑定资源时，用这个静态堆中的描述符来绑定。
这样设计的目的是减少描述符的释放和分配次数，尤其对DX12来说，descriptor handle是从descriptor heap中分配的，多次的释放和分配必然导致堆的碎片化，而使用的bindless后，资源绑定不再强制要求使用的descriptor handles在堆上是连续的，所以使用一个“稳定”的静态堆来实现更加合适。
bindless带来的坏处是多了一次间接寻址，通常需要使用一个常量（或线程id）告诉GPU具体绑定的资源位置，这可能会带来一些性能损失。
## DX12
下面是Atom中使用DX12实现bindless的具体方案：
### resource view
所有view创建时，都需要初始化所有需要用的描述符句柄。
```cpp
        RHI::ResultCode BufferView::InitInternal(RHI::Device& deviceBase, const RHI::DeviceResource& resourceBase)
        {
            Device& device = static_cast<Device&>(deviceBase);

            const Buffer& buffer = static_cast<const Buffer&>(resourceBase);
            const RHI::BufferViewDescriptor& viewDescriptor = GetDescriptor();
            DescriptorContext& descriptorContext = device.GetDescriptorContext();

            // By default, if no bind flags are specified on the view descriptor, attempt to create all views that are compatible with the underlying buffer's bind flags
            // If bind flags are specified on the view descriptor, only create the views for the specified bind flags.
            bool hasOverrideFlags = viewDescriptor.m_overrideBindFlags != RHI::BufferBindFlags::None;
            const RHI::BufferBindFlags bindFlags = hasOverrideFlags ? viewDescriptor.m_overrideBindFlags : buffer.GetDescriptor().m_bindFlags;

            m_memory = buffer.GetMemoryView().GetMemory();
            m_gpuAddress = buffer.GetMemoryView().GetGpuAddress() + viewDescriptor.m_elementOffset * viewDescriptor.m_elementSize;

            if (RHI::CheckBitsAny(bindFlags, RHI::BufferBindFlags::ShaderRead | RHI::BufferBindFlags::RayTracingAccelerationStructure))
            {
                descriptorContext.CreateShaderResourceView(buffer, viewDescriptor, m_readDescriptor, m_staticReadDescriptor);
            }

            if (RHI::CheckBitsAny(bindFlags, RHI::BufferBindFlags::ShaderWrite))
            {
                descriptorContext.CreateUnorderedAccessView(
                    buffer, viewDescriptor, m_readWriteDescriptor, m_clearDescriptor, m_staticReadWriteDescriptor);
            }

            if (RHI::CheckBitsAny(bindFlags, RHI::BufferBindFlags::Constant))
            {
                descriptorContext.CreateConstantBufferView(buffer, viewDescriptor, m_constantDescriptor, m_staticConstantDescriptor);
            }

            return RHI::ResultCode::Success;
        }
```
以创建SRV为例，
```cpp
descriptorContext.CreateShaderResourceView(buffer, viewDescriptor, m_readDescriptor, m_staticReadDescriptor);
```
m_readDescriptor是SRV的描述符，它创建在一个D3D12_DESCRIPTOR_HEAP_FLAG_NONE的堆上，后面绑定是将其复制到D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE堆。用于通常的资源绑定方式。
m_staticReadDescriptor是为bindless创建的句柄，它创建在D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE，并且不会发生改变，可以使用GetBindlessReadIndex接口获取它。
### ShaderResourceGroupPool
Atom中的ShaderResourceGroupPool用于绑定所有更新shader资源。shader资源的布局在根签名中已经声明，这里会在D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE堆上创建句柄，然后将view的句柄复制过来，以保证绑定时描述符句柄是连续的。
下面是其中的部分代码：
```cpp
// process buffer unbounded arrays
for (const RHI::ShaderInputBufferUnboundedArrayDescriptor& shaderInputBufferUnboundedArray : groupLayout.GetShaderInputListForBufferUnboundedArrays())
{
    const RHI::ShaderInputBufferUnboundedArrayIndex bufferUnboundedArrayInputIndex(shaderInputIndex);
    uint32_t tableIndex = shaderInputIndex * RHI::Limits::Device::FrameCountMax + group.m_compiledDataIndex;
    ShaderResourceGroupCompiledData& compiledData = group.m_compiledData[group.m_compiledDataIndex];

    AZStd::span<const RHI::ConstPtr<RHI::DeviceBufferView>> bufferViews = groupData.GetBufferViewUnboundedArray(bufferUnboundedArrayInputIndex);

    // resize the descriptor table allocation if necessary
    if (group.m_unboundedDescriptorTables[tableIndex].GetSize() != bufferViews.size())
    {
        if (group.m_unboundedDescriptorTables[tableIndex].IsValid())
        {
            m_descriptorContext->ReleaseDescriptorTable(group.m_unboundedDescriptorTables[tableIndex]);
            group.m_unboundedDescriptorTables[tableIndex] = DescriptorTable{};
        }

        if (!bufferViews.empty())
        {
            group.m_unboundedDescriptorTables[tableIndex] = m_descriptorContext->CreateDescriptorTable(
                D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV, static_cast<uint32_t>(bufferViews.size()));
            
            if (!group.m_unboundedDescriptorTables[tableIndex].IsValid())
            {
                // It is possible to run out of number of descriptors in the descriptor heap if you are using custom SRG
                // with an unbounded array as it can fragment over time. Consider using Bindless SRG's unbounded arrays as
                // they do not fragment.
                AZ_Assert(
                    false,
                    "Descriptor heap ran out of memory. Please consider increasing number of handles allowed for the "
                    "second value of DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV within platformlimits.azasset file for dx12.");
                return;
            }

            compiledData.m_gpuUnboundedArraysDescriptorHandles[shaderInputIndex] = m_descriptorContext->GetGpuPlatformHandleForTable(group.m_unboundedDescriptorTables[tableIndex]);
        }
    }
    
    const DescriptorTable descriptorTable(group.m_unboundedDescriptorTables[tableIndex].GetOffset(), static_cast<uint16_t>(bufferViews.size()));
    UpdateUnboundedBuffersDescTable(descriptorTable, groupData, shaderInputIndex, shaderInputBufferUnboundedArray.m_access);
    ++shaderInputIndex;
}
```
这里如果绑定的bufferViews数量与上一次不相同，将会触发Descriptor table的释放和重新分配，分配时调用的函数CreateDescriptorTable是在D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE堆上分配了句柄。然后其句柄存入了compiledData中。
一个ShaderResourceGroup可以包含多个无界的buffer view，它们会保存在一个连续的table中，所以这里使用tableIndex来进行索引。
后面的UpdateUnboundedBuffersDescTable就是将ShaderResourceGroup携带的buffer view写入刚刚分配出来的Descriptor table里。
### CommandList
这是对ID3D12GraphicsCommandList的封装。
初始化时，自动绑定描述符堆（Copy队列的CommandList不需要此步骤），
```cpp
        void DescriptorContext::SetDescriptorHeaps(ID3D12GraphicsCommandList* commandList) const
        {
            ID3D12DescriptorHeap* heaps[2];
            heaps[0] = GetPool(D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV, D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE).GetPlatformHeap();
            heaps[1] = GetPool(D3D12_DESCRIPTOR_HEAP_TYPE_SAMPLER, D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE).GetPlatformHeap();
            commandList->SetDescriptorHeaps(2, heaps);
        }
```
可以看到这里绑定的都是SHADER_VISIBLE的。

绑定的函数CommitShaderResources代码片段：
```

```

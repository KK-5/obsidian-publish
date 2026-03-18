RenderAttachmentLayout是Atom中定义渲染附件信息的数据，vulkan中RenderPass和SubPass的创建需要此数据。DX12不会使用这个数据（除了RTV格式），它的”渲染附件“是在记录命令时动态绑定的，也没有RenderPass和SubPass这个概念。
# 结构体层级与关系
RenderAttachmentConfiguration（配置：用哪个 subpass）
    └── RenderAttachmentLayout（整个 render pass 的布局）
            ├── m_attachmentFormats[]     → 所有附件的格式
            └── m_subpassLayouts[]        → 每个 subpass 的布局
                    └── SubpassRenderAttachmentLayout（单个 subpass）
                            ├── m_rendertargetDescriptors[]   → RenderAttachmentDescriptor[]
                            ├── m_subpassInputDescriptors[]   → SubpassInputDescriptor[]
                            ├── m_depthStencilDescriptor      → RenderAttachmentDescriptor
                            └── m_shadingRateDescriptor       → RenderAttachmentDescriptor
# RenderAttachmentDescriptor
```cpp
    //! Describes one render attachment that is part of a layout.
    struct ATOM_RHI_REFLECT_API RenderAttachmentDescriptor
    {
        AZ_TYPE_INFO(RenderAttachmentDescriptor, "{2855E1D2-BDA1-45A8-ABB9-5D8FB1E78EF4}");
        static void Reflect(ReflectContext* context);
        bool IsValid() const;

        bool operator==(const RenderAttachmentDescriptor& other) const;
        bool operator!=(const RenderAttachmentDescriptor& other) const;
        bool IsEqual(const RenderAttachmentDescriptor& other, const bool compareLoadStoreAction) const;

        //! Position of the attachment in the layout.
        uint32_t m_attachmentIndex = InvalidRenderAttachmentIndex;
        //! Position of the resolve attachment in layout (if it resolves).
        uint32_t m_resolveAttachmentIndex = InvalidRenderAttachmentIndex;
        //! Load and store action of the attachment.
        AttachmentLoadStoreAction m_loadStoreAction = AttachmentLoadStoreAction();
        //! The following two flags are only relevant when there are more than one subpasses
        //! that will be merged.
        //! The scope attachment access as defined in the pass template, which will be used
        //! to accurately define the subpass dependencies.
        AZ::RHI::ScopeAttachmentAccess m_scopeAttachmentAccess = AZ::RHI::ScopeAttachmentAccess::Unknown;
        //! The scope attachment stage as defined in the pass template, which will be used
        //! to accurately define the subpass dependencies.
        AZ::RHI::ScopeAttachmentStage m_scopeAttachmentStage = AZ::RHI::ScopeAttachmentStage::Uninitialized;
        //! Extra data that can be passed for platform specific operations.
        RenderAttachmentExtras* m_extras = nullptr;
    };
```

描述一个附件在某个 subpass 中的用法（颜色/深度/解析/着色率等）。

| 成员                         | 含义                     | Vulkan 大致对应                                                      |
| -------------------------- | ---------------------- | ---------------------------------------------------------------- |
| `m_attachmentIndex`        | 该附件在全局附件列表中的索引         | `VkAttachmentReference.attachment`                               |
| `m_resolveAttachmentIndex` | 解析目标附件索引（MSAA resolve） | `VkSubpassDescription.pResolveAttachments[]` 中的 attachment index |
| `m_loadStoreAction`        | Load/Store 行为          | `VkAttachmentDescription.loadOp/storeOp`（以及 stencil）             |
| `m_scopeAttachmentAccess`  | 读/写访问                  | 用于推导 `VkSubpassDependency` 的 `srcAccessMask/dstAccessMask`       |
| `m_scopeAttachmentStage`   | 使用阶段                   | 用于推导 `VkSubpassDependency` 的 `srcStageMask/dstStageMask`         |
| `m_extras`                 | 平台扩展                   | 可映射到 Vulkan 扩展结构                                                 |

# SubpassInputDescriptor
```cpp
    //! Describes a subpass input attachment.
    struct ATOM_RHI_REFLECT_API SubpassInputDescriptor
    {
        AZ_TYPE_INFO(SubpassInputDescriptor, "{5E5B933D-8209-4722-8AC5-C3FEA1D75BB3}");
        static void Reflect(ReflectContext* context);

        bool operator==(const SubpassInputDescriptor& other) const;
        bool operator!=(const SubpassInputDescriptor& other) const;

        //! Attachment index that this subpass input references.
        uint32_t m_attachmentIndex = 0;
        //! Aspects that are used by the input (needed by some implementations, like Vulkan, when creating a renderpass with a subpass input)
        RHI::ImageAspectFlags m_aspectFlags = RHI::ImageAspectFlags::None;
        //! The following two flags are only relevant when there are more than one subpasses
        //! that will be merged.
        //! The scope attachment access as defined in the pass template, which will be used
        //! to accurately define the subpass dependencies.
        AZ::RHI::ScopeAttachmentAccess m_scopeAttachmentAccess = AZ::RHI::ScopeAttachmentAccess::Unknown;
        //! The scope attachment stage as defined in the pass template, which will be used
        //! to accurately define the subpass dependencies.
        AZ::RHI::ScopeAttachmentStage m_scopeAttachmentStage = AZ::RHI::ScopeAttachmentStage::Uninitialized;
        //! Load and store action of the attachment.
        AttachmentLoadStoreAction m_loadStoreAction = AttachmentLoadStoreAction();
        //! Extra data that can be passed for platform specific operations.
        RenderAttachmentExtras* m_extras = nullptr;
    };
```
描述本 subpass 内以“input attachment”方式引用的附件（只读，通常用于同一 render pass 内前一 subpass 的输出）。

|成员|含义|Vulkan 大致对应|
|---|---|---|
|`m_attachmentIndex`|引用的附件在全局列表中的索引|`VkAttachmentReference.attachment`（在 `pInputAttachments` 中）|
|`m_aspectFlags`|使用的 image aspect|`VkAttachmentReference.aspectMask`（Vulkan 创建带 input attachment 的 render pass 时需要）|
|`m_scopeAttachmentAccess/Stage`|访问与阶段|用于 subpass dependency 的 access/stage mask|
|`m_loadStoreAction`|Load/Store|对应该附件的 load/store|
|`m_extras`|平台扩展|同上|
# SubpassRenderAttachmentLayout
描述一个 subpass 用到的所有附件种类和数量。
```cpp
    //! Describes the attachments of one subpass as part of a render target layout.
    //! It include descriptions about the render targets, subpass inputs and depth/stencil attachment.
    struct ATOM_RHI_REFLECT_API SubpassRenderAttachmentLayout
    {
        AZ_TYPE_INFO(SubpassRenderAttachmentLayout, "{7AF04EC1-D835-4F97-8433-0D445C0D6F5B}");
        static void Reflect(ReflectContext* context);

        bool operator==(const SubpassRenderAttachmentLayout& other) const;
        bool operator!=(const SubpassRenderAttachmentLayout& other) const;
        bool IsEqual(const SubpassRenderAttachmentLayout& other, const bool compareLoadStoreAction) const;

        //! Number of render targets in the subpass.
        uint32_t m_rendertargetCount = 0;
        //! Number of subpass input attachments in the subpass.
        uint32_t m_subpassInputCount = 0;
        //! List of render targets used by the subpass.
        AZStd::array<RenderAttachmentDescriptor, Limits::Pipeline::AttachmentColorCountMax> m_rendertargetDescriptors = { {} };
        //! List of subpass inputs used by the subpass.
        AZStd::array<SubpassInputDescriptor, Limits::Pipeline::AttachmentColorCountMax> m_subpassInputDescriptors = { {} };
        //! Descriptor of the depth/stencil attachment. If not used, the attachment index is InvalidRenderAttachmentIndex.
        RenderAttachmentDescriptor m_depthStencilDescriptor;
        //! Descriptor of the shading rate attachment. If not used, the attachment index is InvalidRenderAttachmentIndex.
        RenderAttachmentDescriptor m_shadingRateDescriptor;
    };
```

|成员|含义|Vulkan 大致对应|
|---|---|---|
|`m_rendertargetCount`|颜色附件数量|`VkSubpassDescription.colorAttachmentCount`|
|`m_rendertargetDescriptors`|颜色（及 resolve）附件|`pColorAttachments` / `pResolveAttachments`|
|`m_subpassInputCount`|Input 附件数量|`inputAttachmentCount`|
|`m_subpassInputDescriptors`|Input 附件|`pInputAttachments`|
|`m_depthStencilDescriptor`|深度/模板|`pDepthStencilAttachment`|
|`m_shadingRateDescriptor`|着色率附件|`VK_KHR_fragment_shading_rate` 的 attachment|

用途：对应 `VkSubpassDescription`（不含 dependency，dependency 由 access/stage 等信息在别处推导）。
# RenderAttachmentLayout（整个 Render Pass 的布局）
描述整个 render pass：有多少个附件、每个附件的格式、有多少个 subpass、每个 subpass 的布局。
```cpp
    //! A RenderAttachmentLayout consist of a description of one or more subpasses.
    //! Each subpass is a collection of render targets, subpass inputs and depth stencil attachments.
    //! Each subpass corresponds to a phase in the rendering of a Pipeline.
    //! Subpass outputs can be read by other subpasses as inputs.
    //! A RenderAttachmentLayout may be implemented by the platform using native functionality, achieving a
    //! performance gain for that specific platform. On other platforms, it may be emulated to achieve the same result
    //! but without the performance benefits. For example, Vulkan supports the concept of subpass natively.
    class ATOM_RHI_REFLECT_API RenderAttachmentLayout
    {
    public:
        AZ_TYPE_INFO(RenderAttachmentLayout, "{5ED96982-BFB0-4EFF-ABBE-1552CECE707D}");
        static void Reflect(ReflectContext* context);
  
        HashValue64 GetHash() const;

        bool operator==(const RenderAttachmentLayout& other) const;
        bool IsEqual(const RenderAttachmentLayout& other, const bool compareLoadStoreAction) const;

        //! Number of total attachments in the list.
        uint32_t m_attachmentCount = 0;
        //! List with all attachments (renderAttachments, resolveAttachments and depth/stencil attachment).
        AZStd::array<Format, Limits::Pipeline::RenderAttachmentCountMax> m_attachmentFormats = { {} };

        //! Number of subpasses.
        uint32_t m_subpassCount = 0;
        //! List with the layout of each subpass.
        AZStd::array<SubpassRenderAttachmentLayout, Limits::Pipeline::SubpassCountMax> m_subpassLayouts;
    };
```

|成员|含义|Vulkan 大致对应|
|---|---|---|
|`m_attachmentCount`|附件总数|`VkRenderPassCreateInfo.attachmentCount`|
|`m_attachmentFormats`|每个附件的格式|`VkAttachmentDescription[].format`（其他 loadOp/storeOp 等可从各 subpass 的 descriptor 汇总）|
|`m_subpassCount`|Subpass 数量|`VkRenderPassCreateInfo.subpassCount`|
|`m_subpassLayouts`|每个 subpass 的布局|`VkRenderPassCreateInfo.pSubpasses` → 每个 `VkSubpassDescription`|

用途：对应 `VkRenderPassCreateInfo` 的主干数据（attachment 列表 + subpass 数组）。
# RenderAttachmentConfiguration（“当前用哪个 Subpass”的配置）
在已有 `RenderAttachmentLayout`（可能包含多个 subpass）的基础上，表示当前管线/Pass 实际使用的是哪一个 subpass，并对外提供该 subpass 的格式、数量等查询接口。
```cpp
    //! Describes the layout of a collection of subpasses and it defines which of the subpasses this
    //! configuration will be using.
    struct ATOM_RHI_REFLECT_API RenderAttachmentConfiguration
    {
        AZ_TYPE_INFO(RenderAttachmentConfiguration, "{037F27A5-B568-439B-81D4-928DFA3A74F2}");
        static void Reflect(ReflectContext* context);

        HashValue64 GetHash() const;

        bool operator==(const RenderAttachmentConfiguration& other) const;
        bool IsEqual(const RenderAttachmentConfiguration& other, const bool compareLoadStoreAction) const;

        //! Returns the format of a render target in the subpass being used.
        Format GetRenderTargetFormat(uint32_t index) const;
        //! Returns the format of a subpass input in the subpass being used.
        Format GetSubpassInputFormat(uint32_t index) const;
        //! Returns the format of resolve attachment in the subpass being used.
        Format GetRenderTargetResolveFormat(uint32_t index) const;
        //! Returns the format of a depth/stencil in the subpass being used.
        //! Will return Format::Unkwon if the depth/stencil is not being used.
        Format GetDepthStencilFormat() const;
        //! Returns the number of render targets in the subpass being used.
        uint32_t GetRenderTargetCount() const;
        //! Returns the number of subpass inputs in the subpass being used.
        uint32_t GetSubpassInputCount() const;
        //! Returns true if the render target is resolving in the subpass being used.
        bool DoesRenderTargetResolve(uint32_t index) const;

        //! Layout of the render target attachments.
        RenderAttachmentLayout m_renderAttachmentLayout;
        //! Index of the subpass being used.
        uint32_t m_subpassIndex = 0;
    };
```

|成员|含义|Vulkan 对应|
|---|---|---|
|`m_renderAttachmentLayout`|完整 render pass 布局|同上，用于创建 `VkRenderPass`|
|`m_subpassIndex`|当前使用的 subpass 索引|创建 `VkGraphicsPipeline` 时传入的 `subpass` 索引|

`GetRenderTargetFormat`、`GetDepthStencilFormat`、`GetRenderTargetCount` 等接口都是针对 `m_subpassLayouts[m_subpassIndex]` 的便捷访问，便于管线和 framebuffer 绑定阶段使用。
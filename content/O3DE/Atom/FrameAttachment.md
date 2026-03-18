# FrameAttachment
```cpp
		// 一个哈希字符串
        AttachmentId m_attachmentId;
        
        // 引用的资源
        Ptr<Resource> m_resource;
        
        // Imported和Transient两种生命周期，默认是Imported
        AttachmentLifetimeType m_lifetimeType;
        HardwareQueueClassMask m_usedQueueMask = HardwareQueueClassMask::None;
        HardwareQueueClassMask m_supportedQueueMask = HardwareQueueClassMask::None;
        // we need to store the first device this frame attachment is used on to initialize the clear value
        int m_firstDeviceIndex{ MultiDevice::InvalidDeviceIndex };
        // key是device index
        AZStd::unordered_map<int, ScopeInfo> m_scopeInfos;
        
        // ScopeInfo定义，记录此FrameAttachment使用的第一个和最后一个Scope
        struct ScopeInfo
        {
            ScopeAttachment* m_firstScopeAttachment{ nullptr };
            ScopeAttachment* m_lastScopeAttachment{ nullptr };
            Scope* m_firstScope{ nullptr };
            Scope* m_lastScope{ nullptr };
        };
```
## BufferFrameAttachment
继承自FrameAttachment，多了一个BufferDescriptor。
```cpp
    BufferDescriptor m_bufferDescriptor;
```
构造方式多了一种
```cpp
    /// Initialization for transient buffers.
    BufferFrameAttachment(const TransientBufferDescriptor& descriptor);
    
    // 包装AttachmentId和BufferDescriptor
    struct ATOM_RHI_REFLECT_API TransientBufferDescriptor
    {
        TransientBufferDescriptor() = default;

        TransientBufferDescriptor(
            const AttachmentId& attachmentId,
            const BufferDescriptor& bufferDescriptor);

        /// The attachment id to associate with the transient buffer.
        AttachmentId m_attachmentId;

        /// The buffer descriptor used to create the transient buffer.
        BufferDescriptor m_bufferDescriptor;

        HashValue64 GetHash(HashValue64 seed = HashValue64{ 0 }) const;
    };
```
## ImageFrameAttachment
和BufferFrameAttachment相似，额外包含一个ImageDescriptor。
```cpp
        ImageDescriptor m_imageDescriptor;

        /// TODO: Replace with optional. A user clear value in the case of a transient attachment.
        bool m_hasClearValueOverride = false;
        ClearValue m_clearValueOverride;
```
# ScopeAttachment
```cpp
    //! A ScopeAttachment is created when a FrameAttachment is "attached" to a specific scope. A single FrameAttachment
    //! exists for a given AttachemntId, but many ScopeAttachments can exist as "children" of a FrameAttachment. More
    //! precisely, ScopeAttachment forms a linked list, where the first node is the first "usage" on the first scope, and the last node
    //! the last usage on the last scope. FrameAttachment references the head and tail of this linked list.
    //!
    //! The FrameAttachment owns the Attachment instance (i.e. the actual resource). The ScopeAttachment owns a view into that
    //! resource. A scope is able to utilize the view during compilation and execution. 
```
ScopeAttachment和FrameAttachment就像Resource和ResourceVIew的关系，一个FrameAttachment可能会对应多个ScopeAttachment，并且FrameAttachment持有一个链表，保存了自己的所有ScopeAttachment。
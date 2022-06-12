## Proposal
- `MarkDirtyResource` on a buffer if `contents` is called and the returned pointer is not NULL (type should be `MTLStorageModeShared`).
- `MarkDirtyResource` on a buffer if  `didModifyRange` is called (type should be `MTLStorageModeManaged`).
- `MarkDirtyResource` on a buffer if buffer type is `MTLStorageModePrivate`.
- `MTLBuffer` type is only known at creation time.

## Questions
- How do dirty resources interact with frame referenced resources. Will dirty resources always be output or is it only if they are frame referenced also?
  - **TODO**

## Notes
- [Metal Documentation: Choosing a Resource Storage Mode in macOS](https://developer.apple.com/documentation/metal/setting_resource_storage_modes/choosing_a_resource_storage_mode_in_macos?language=objc)
- MTLBuffer Storage [Modes](https://developer.apple.com/documentation/metal/mtlstoragemode?language=objc)
  - `MTLStorageModeShared` : Stored in CPU memory accessible by the GPU.
    - Application manages synchronisation:
    - CPU->GPU: `MTLCommandBuffer::commit`
    - GPU->CPU: when command buffer completes execution ie. `MTLCommandBuffer::status == MTLCommandBufferStatusCompleted`
  - `MTLStorageModeManaged` : Stored in CPU and GPU memory.
    - Metal driver manages synchronisation:
    - CPU->GPU: `MTLBuffer::didModifyRange`
    - GPU->CPU: `MTLBlitCommandEncoder::synchronizeResource`
  - `MTLStorageModePrivate` : Stored in GPU memory.
    - Only accessible by copying to a `MTLStorageModeShared` buffer.
- `MTLBuffer` CPU contents locked / submitted to GPU on `MTLCommandBuffer:: commit`.
- RenderCommandEncoder has APIs to use `MTLBuffer` ie. `setVertexBuffer`, `setFragmentBuffer`.

## Thoughts
- To read buffer data will need to convert `MTLStorageModePrivate` buffers to `MTLStorageModeManaged` or use a temporary intermediate `MTLStorageModeManaged` buffer.
- Mark buffer dirty if `contents` is called and returned pointer is not NULL : type should be `MTLStorageModeShared`
- Mark buffer dirty if  `didModifyRange` is called, type should be `MTLStorageModeManaged`
- Mark buffer dirty if buffer type is `MTLStorageModePrivate` ie. GPU only
- Should `MTLStorageModeManaged` buffers only be tracked if `contents` is called on them?
  - In theory yes if the returned pointer is not NULL.
  - In practice might be easier to snashot the buffer without explicit tracking.
- Should buffers only be tracked if they can reside in CPU memory?
  - GPU only buffers do not need to be tracked for CPU modification. They need to be tracked for GPU modification which is hard to do for all situations.
    - In practice easier to snashot the buffer without explicit tracking.
  - CPU buffers with explicit synchronization can be handled in the serialization of the synchronization APIs
    - In practice might be easier to snashot the buffer without explicit tracking.
- Attach the buffers to the `MTLRenderCommandEncoder` parent `MTLCommandBuffer` in `setVertexBuffer`, `setFragmentBuffer` APIs.
- On `MTLCommandBuffer::commit` in active capture serialize attached buffer contents.
- On `MTLCommandBuffer::commit` replay de-serialize buffer contents.
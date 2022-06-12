## Proposed Solution
#### `MTLDevice`
- No parent.
- Manually marked as frame referenced in `StartFrameCapture`.

#### `MTLLibrary`
- No parent.
- Not marked as frame referenced.

#### `MTLFunction`
- Parent is set as `MTLLibary`.
- Not marked as frame referenced.

#### `MTLCommandQueue`
- No parent.
- Marked as frame referenced during active capture when a child `MTLCommandBuffer` calls `commit` API.

#### `MTLCommandBuffer`
- No parent (logical parent is `MTLCommandQueue`, this is not set as `MTLCommandQueue` is manually marked as frame referenced by its child `MTLCommandBuffer`).
- Manually added to capture, stored in a list when `commit` is called during active capture.

#### `MTLRenderPipelineState`
- Any `MTLFunction`(s) referenced by its descriptor are set as parents in creation APIs ie. `MTLDevice::newRenderPipelineStateWithDescriptor`.
- Marked as frame referenced during active capture by APIs which use it ie. `MTLRenderCommandEncoder::setRenderPipelineState`.

#### `MTLRenderCommandEncoder`
- No parent.
- Not marked as frame referenced.
- Its creation and API chunks are added to its parent `MTLCommandBuffer` record.

#### `MTLTexture`
- No parent.
1. Marked as frame referenced during capture by APIs which use it ie. `MTLCommandBuffer::renderCommandEncoderWithDescriptor`, 
`MTLRenderCommandEncoder::setFragmentTexture`.
- **INCOMPLETE: describe marking dirty.**

#### `MTLBuffer`
- No parent.
- Marked as frame referenced during capture by APIs which use it ie. `MTLCommandBuffer::renderCommandEncoderWithDescriptor`, `MTLRenderCommandEncoder::setVertexBuffer`, `MTLRenderCommandEncoder::setFragmentBuffer`.
- **INCOMPLETE: describe marking dirty.**

## Primary Questions to Answer
1. Is it worth tracking precise global resource usage tied to the active capture frame ie. `MTLCommandQueue`, `MTLLibrary`, `MTLFunction`, `MTLRenderPipelineState`.
    - For basic usage, this is possible, however, expect it is not viable to achieve this for all scenarios and possible usage.
    - **NOT ANSWERED**
1. The primary rationale for the Permanent resource type is that it is invalid to call the resource creation of these types multiple times.
    - Perhaps the correct approach is to handle it on the replay side and filter these chunks out of the per-frame replayed chunks?
    - **NOT ANSWERED**

## Supporting Questions to Answer
1. The ultimate goal is for a Permanent Resource created during the active capture frame then its creation chunk(s) appear before the `Frame Section`. Is this goal realistic and achievable?
    - **NOT ANSWERED**
1. Should `AddParent` only be used when resource A creates resource B? Is it valid to use `AddParent` if resource A only contains references to resource B (and resource B might be referenced by other resources)?
    - Note: using `AddParent` to describe resource A references resource B is working.
    - **NOT ANSWERED**

## Experiments to Perform
1. **NONE**

## Results of Experiments 
1. Do not add `MTLCommandQueue` as a parent of `MTLCommandBuffer` (it is added as a parent and marked as frame referenced).
   - Capture output is correct and the `MTLCommandQueue` creation chunk is in the `Initialisation Section`.
1. Check a capture when the application creates the Permanent Resources in the same frame as using them, then the resource creation chunks are output before the `Frame Section`.
    - Works for `MTLCommandQueue`, `MTLRenderPipelineState`, `MTLLibrary`, `MTLFunction`.
1. Solve the problem without using custom code for Permanent Resources and instead use the existing resource manager and record APIs
    - In `MTLDevice::newRenderPipelineState` add the vertex and fragment functions as parents of the `MTLRenderPipelineState` resource.
    - In `MTLCommandBuffer::commit` in an active capture frame mark the parent `MTLCommandQueue` as frame referenced.
    - In `MTLRenderCommandEncoder::setRenderPipelineState` in an active capture frame mark the `MTLRenderPipelineState` as frame referenced.
    - **Doing these changes results in the Permanent Resource chunks being output in the desired place in the capture file ie. before `Frame Section`.**
1. Add PermanentResource array to the wrapped device and then in `EndFrameCapture` mark these resources as FrameReferenced
    - In `MTLDevice::newRenderPipelineState` add the referenced `MTLFunction` as a permananent resource
    - In `MTLRenderCommandEncoder::setRenderPipelineState` add the `MTLRenderPipelineState` as a permanent resource
     - Note: this is not precise tracking, the `MTLRenderPipelineState` is not marked as referenced when its command buffer is committed. The resource might not be required.
     - This should not be required as the resource manager has the systems in place to solve this.
1. Debug why marking resources as frame referenced during `EndFrameCapture` makes the chunks get output before the `Frame Section`
    - They get added to `m_FrameReferencedResources` which are output by `InsertReferencedChunks` which is before the `Frame Section` output.
1. Comment out all marking of resources as referenced. Mark `MTLFunction` as referenced when `MTLRenderPipelineState` is created: see if this marking also brings the `MTLLibrary` creation chunk via the parent relationship.
    - Created does not work, no connection to the command buffer record, no `MTLFunction` or `MTLLibrary` chunks output.
    - Marking `MTLRenderPipelineState` as frame referenced during `set` and `MTLFunction` as frame referenced during creation does not work for `MTLFunction` only the `MTLRenderPipelineState` creation chunks are output. 
    - For `MTLCommandBuffer` marking `MTLCommandQueue` as the parent, then when the `MTLCommandBuffer` chunks are output the `MTLCommandQueue` creation records are **also** output.
1. **INVALID** Mark resources referenced by `MTLRenderPipelineState` during `setRenderPipelineState` but only during active capture.
    - Can't extract from `MTLRenderPipelineState`, only the descriptor has the resource references ie. `MTLFunction`.
    - The tracking would need to be done when `MTLRenderPipelineState` is created.
1. **INVALID** In `EndFrameCapture` follow resource references from the tracked `MTLCommandBuffer`'s to `MTLRenderCommandEncoder` and then `MTLFunction` and possibly `MTLLibrary`
    - Can't extract from `MTLRenderPipelineState`, only the descriptor has the resource references ie. `MTLFunction`. 
    - The tracking would need to be done when `setRenderPipelineState` is created.
1. Is Metal-specific capture code required to achieve the goals of Permanent Resources (should never be recreated) ie. `MTLCommandQueue`, `MTLLibrary`, `MTLFunction`, `MTLRenderPipelineState` etc.
    - Need to complete the active frame to identify any resources used or created during the capture.
    - Adding code in `EndFrameCapture` to manually mark these resources appears to be working. It needs to be done before the capture state transitions back to `BackgroundCapturing`.

## Notes
- `MTLLibrary` creates and is set as parent of `MTLFunction`
- `MTLFunction` is referenced by `MTLRenderPipelineState` resource
- `MTLRenderPipelineState` is referenced by `MTLRenderCommandEncoder` API
- `MTLCommandQueue` creates `MTLRenderCommandEncoder` (parent is not set).
- Resources can have multiple parents.

### List of Resources
- `MTLDevice`
- `MTLLibrary`
- `MTLFunction`
- `MTLCommandQueue`
- `MTLCommandBuffer`
- `MTLRenderPipelineState`
- `MTLTexture`
- `MTLBuffer`
- `MTLRenderCommandEncoder`

## For each Resource explain:
1. Parent/child relationships.
1. For all captures (including first frame captures), irrespective of when the resource is created, which section of the capture file the resource creation chunk is expected to be in:
     - `Initialisation Section` (before chunk `Internal::List of Initial Contents Resources`).
     - `Frame Section` (after chunk `Internal::Beginning of Capture`)
1. If it should be listed in the initial contents resources chunk `Internal::List of Initial Contents Resources`.
1. When and how it should be marked as frame referenced.
1. Immutable or Mutable after it is created.
1. Type of Resource: Permanent (should never be recreated), Transitory.
1. Notes.

#### `MTLDevice`
1. Has no parents, in theory, is the ultimate parent of all resources.
1. Present in `Initialisation Section`.
1. Not listed in the initial contents chunk.
1. Manually marked as frame referenced in `StartFrameCapture`.
1. Immutable, serialized content doesn't change after creation.
1. Permanent Resource.
1. ID is serialized in chunk `MTLCreateSystemDefaultDevice`.

#### `MTLLibrary`
1. Logical parent is `MTLDevice`, not set in practice.
1. Present in `Initialisation Section`.
1. Not listed in the initial contents chunk.
1. Not marked as frame referenced.
1. Immutable, serialized content doesn't change after creation.
1. Permanent Resource.

#### `MTLFunction`
1. Parent is set as `MTLLibary`.
1. Present in `Initialisation Section`.
1. Not listed in the initial contents chunk.
1. Not marked as frame referenced.
1. Immutable, serialized content doesn't change after creation.
1. Permanent Resource.
1. Set as a parent of `MTLRenderPipelineState` in `MTLDevice::newRenderPipelineState`.

#### `MTLCommandQueue`
1. Logical parent is `MTLDevice`, not set in practice.
1. Present in `Initialisation Section`.
1. Not listed in the initial contents chunk.
1. Marked as frame referenced during active capture when a child `MTLCommandBuffer` calls `commit` API.
1. Immutable, serialized content doesn't change after creation.
1. Permanent Resource.

#### `MTLCommandBuffer`
1. Parent is set as `MTLCommandQueue`.
1. Present in `Frame Section`.
1. Not listed in the initial contents chunk.
1. Manually added to capture, stored in a list when `commit` is called during active capture.
1. Mutable.
1. Transitory Resource

#### `MTLRenderPipelineState`
1. Logical parent is `MTLDevice`, not set in practice. The `MTLFunction`(s) referenced by its descriptor are set as parents.
1. Present in `Initialisation Section`.
1. Not listed in the initial contents chunk.
1. Marked as frame referenced during active capture by APIs which use it ie. `MTLRenderCommandEncoder::setRenderPipelineState`.
1. Immutable, serialized content doesn't change after creation.
1. Permanent Resource.

#### `MTLTexture`
1. Logical parent is `MTLDevice`, not set in practice.
1. Present in `Initialisation Section`.
1. Listed in the initial contents chunk.
1. Marked as frame referenced during capture by APIs which use it ie. `MTLCommandBuffer::renderCommandEncoderWithDescriptor`, 
`MTLRenderCommandEncoder::setFragmentTexture`.
1. Mutable.
1. Permanent Resource.

#### `MTLBuffer`
1. Logical parent is `MTLDevice`, not set in practice.
1. Present in `Initialisation Section`.
1. Listed in the initial contents chunk.
1. Marked as frame referenced during capture by APIs which use it ie. `MTLCommandBuffer::renderCommandEncoderWithDescriptor`, `MTLRenderCommandEncoder::setVertexBuffer`, `MTLRenderCommandEncoder::setFragmentBuffer`.
1. Mutable.
1. Permanent Resource.

#### `MTLRenderCommandEncoder`
1. Logical parent is `MTLCommandBuffer`, not set in practice.
1. Present in `Frame Section`.
1. Not listed in the initial contents chunk.
1. Not marked as frame referenced.
1. Mutable.
1. Transitory Resource.
1. Its creation and API chunks are added to its parent `MTLCommandBuffer` record.
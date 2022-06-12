
`bool WrappedVulkan::Serialise_vkCmdBlitImage`

```
if(IsReplayingAndReading())
{
    if(IsActiveReplaying(m_State))
    {
        ....
    }
    else
    {
        AddAction(action);

        VulkanActionTreeNode &actionNode = GetActionStack().back()->children.back();

        if(srcImage == destImage)
        {
          actionNode.resourceUsage.push_back(make_rdcpair(
              GetResID(srcImage), EventUsage(actionNode.action.eventId, ResourceUsage::Resolve)));
        }
        else
        {
          actionNode.resourceUsage.push_back(make_rdcpair(
              GetResID(srcImage), EventUsage(actionNode.action.eventId, ResourceUsage::ResolveSrc)));
          actionNode.resourceUsage.push_back(make_rdcpair(
              GetResID(destImage), EventUsage(actionNode.action.eventId, ResourceUsage::ResolveDst)));
        }
    }
}
```

What about a linked list for each resourceId
```
uint32_t StartEID
uint16_t DeltaEID
uint16_t ResourceUsage
```

A million resources => 8MB for the heads
5 stage changes per resource -> 40MB

## Capture
* Resource ID : nothing else. It is the active ID and serialised ID.

## Replay
* Resource ID : can be a original ID (serialised ID) or a live ID or the active ID of a created resource.

**Replay IDs start from 1000000000000000000.**

* `LiveID` : go from a serialised ID to active replay ID : `original -> live` ie. `GetLiveID(originalID)`.
* `OriginalID` : go from a live replay ID to a serialised ID : `live -> original` ie. `GetOriginalID(liveID)`.
* `ReplacedID` : replaces the resource of an original ID with a new replay ID : `replacement -> original`.
  * Used for Shader Editiing.
  * Does not affect `GetOriginalID()` use `GetUnreplacedOriginalID()` instead.
  * Does affect `GetLiveID()`.
* `GetUnreplacedOriginalID()` : get the `original ID` for a `real ID` that may be a replacement.  i.e. 
  * original ID **123** is live ID **10000000000000000005**
  * live ID **10000000000000000005** is replaced with **10000000000000000039**
  * calling `GetUnreplacedOriginalID` with **10000000000000000005** or **10000000000000000039** will return ID **123**.
* In theory could have multiple replacements for the same original ID
  * No asserts if trying to replace an original ID which already has a replacement.
  * Not defined which replacement ID would get used.

## TODO

## Questions
* Are replacements usually temporary ie. to support custom replay rendering?
* When should `ReplaceResource()` be used during replay or serialised APIs?
* How to work with replay resources which are recreated during replay ie. RenderPass's?

## Thoughts
* Could add asserts and perhaps parameter/variable name changes in replacement methods to prevent accidental misuse or misunderstanding of the methods.
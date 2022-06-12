## Capture
Resource ID : nothing else. It is the active ID and serialised ID.

## Replay
Resource ID : can be a serialised capture ID or the active ID of a created resource.

**Replay IDs start from 1000000000000000000.**

`LiveID` : go from a serialised ID to active replay ID : `original -> live` ie. `GetLiveID(originalID)`.

`OriginalID` : go from an active replay ID to a serialised ID : `live -> original` ie. `GetOriginalID(liveID)`.

`ReplacedID` : replace an active replay ID with a new replay ID : `live -> replacement`.
* Does not affect `GetOriginalID`.
* Does affect `GetLiveID`.

`GetUnreplacedOriginalID()` : get the `original ID` for a `real ID` that may be a replacement.  i.e. 
* original ID **123** is live ID 10000000000000000005
* live ID 10000000000000000005 is replaced with 10000000000000000039
* calling `GetUnreplacedOriginalID` with 10000000000000000005 or 10000000000000000039 will return ID **123**.

## Questions
* Are replacements usually temporary ie. support custom replay rendering?
* When should `ReplaceResource` be used during replay or serialised APIs?
* UnreplacedOriginalID 
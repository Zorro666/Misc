## Potential Logically Independent Succesive Commits
1. Base Pieces
	1. Global Scope Hooks ie. create device
	1. Obj-C & Metal ID wrapped structs helper macros
	1. helper structs (no wrapping or hooking)
	1. almost no obj-c code, perhaps 1 .mm file
1. Capture base support
	1. hooked Metal Objects
	1. no replay support (all stubs)
	1. a lot of .mm files
1. Structured file support in replay
	1. convert to XML
	1. no .mm files 
3. Replay base support including QRenderDoc files/code
	1. Pipeline State UI in basic state
	1. Texture Viewer in basic state
	1. no .mm files

## PRs
- Could be done as unique PRs or a single PR

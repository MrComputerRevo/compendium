# Thread management

The following information is based on a reference implementation that included function symbols (C;C Android) and uses official namings for data structures.

## Thread creation

All created threads are kept in an array `tagTHREAD ThdBuf[100]`.

Next free thread address in the buffer is kept in a global variable `NextFreeThread`.

Upon creation if `NextFreeThread` is null, `THDcreate(int group)` returns null, which in turn will cause an
unhandled exception, since in the reference implementation it is not checked if the created thread is null or not.

`tagTHREAD ThdIndex[24]` consists of 12 groups of dummy threads that are linked between themselves (a total of 24 threads).
When a new thread is created it is kept in `ThdBuf` but is linked in
between the dummy threads in `ThdIndex` with specified group.

Each group is linked individually and is not linked with any other groups.

## Thread control

`ThreadControl` instruction (TODO: link) controls the whole group, the flag values for whole groups 
are kept in `int ThdControl[12]`. This way the whole group can be paused, destroyed, displayed, etc.

Possible flag values for the reference implementation (citation needed):

```C
enum THD_FLAG_STATE
{
  FLG_DESTROY = 0x8000000, //The thread will be destroyed on the next frame
  FLG_DISP = 0x20000000, //The thread will be drawn on the next frame
  FLG_NONE = 0x0, 
  FLG_PAUSE = 0x40000000, //The thread will not be executed on the next frame, but may be drawn if the FLG_DISP is set
};
```

## Thread execution

When running the threads with `THDexec()` a `tagTHREAD* THDtable[100]` is populated based on flag values
in `ThdControl`. Only threads with no `FLG_PAUSE` or `FLG_DESTROY` set are added to the array. At this point, whole groups or individual threads can be destroyed, but otherwise groups
don't matter for execution.

After `THDtable` is built, it is sorted by priority values stored in `THD_EXEC_PRI` from lowest to highest
(in the reference implementation `THD_EXEC_PRI` for all threads is actually set to 0). Once again, groups don't play a role at this point, `THDtable` is simply an array of thread pointers.

After `THDtable` is sorted it is executed using `SCRexec(tagTHREAD*)`. After execution `THDexec()` function loops through `THDtable` again
and destroys the threads that got the `FLG_DESTROY` flag set.

## Thread drawing

When drawing the threads with `THDdrawAll()`, `tagTHREAD* THDtable[100]` is populated with threads that have
`FLG_DISP` set and for which `THD_DRAW_PRI` is more or equal to `PRImask`, a priority mask, which is set by an
instruction (TODO: link) and is 0 by default in the reference implementation.

After `THDtable` is built, it is sorted by priority values stored in `THD_DRAW_PRI` from lowest to highest.

After `THDtable` is sorted the threads trigger a drawing function based on `THD_DRAW_TYPE` value.

In the reference implementation (Chaos;Child Android) the possible values are:

```C
enum DRAWTYPE
{
  DRAWTYPE_TEXT = 0x0, //Draws the main story text as well as the message window, wait icon and tips pop-up.
  DRAWTYPE_MAIN = 0x1, //Draws all the BGs, characters, masks, effects, screen captures, movies, etc. 
  DRAWTYPE_MASK = 0x3, //Draws a full screen mask with color value taken from ThreadVars[0] and alpha value from ScrWork[3318].
  DRAWTYPE_SYSTEMTEXT = 0x4, //Draws a system selection window, not actually seen under normal circumstances in the reference implementation (not to be confused with system message)
  DRAWTYPE_MAINCHIP = 0x6, //TODO
  DRAWTYPE_SYSMENU = 0xA, //Draws the title menu, and all other system menus
  DRAWTYPE_SYSTEMMES = 0xB, //Draws the system message window
  DRAWTYPE_PLAYDATA = 0xC, //TODO
  DRAWTYPE_MUSICMODE = 0xE, //TODO
  DRAWTYPE_MOVIEMODE = 0x10, //TODO
  DRAWTYPE_SAVEICON = 0x12, //Draws the quick save icon
  DRAWTYPE_GLOBALSYSMES = 0x15 //TODO
  DRAWTYPE_DEBUGEDITER = 0x1E, //Draws the debug editor which allows displaying and editing of FlagWork and ScrWork, provided worklist.txt is present.
};
```

After drawing of all the threads is finished, flag `SF_SAVECAPTURE` is set to 0.

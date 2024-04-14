# Performance Tuning in HaxeFlixel

Initial notes for performance tuning. I hope to fill this out over time with references, tools and tips and techniques. First up is haxe-instrument.

## Haxe-Instrument

Haxe-instrument, aka the `instrument` haxelib is a macro based instrumenting profiler. Details are available at https://github.com/AlexHaxe/haxe-instrument.

To set this up for flixel you need currently at any rate to do a couple of things.

   1. install the latest version not the haxelib version
   ```
   haxelib git instrument https://github.com/AlexHaxe/haxe-instrument.git
   ```
   2. There are some issues with compilation because `instrument` modifies code, so it can cause compilation issues if it introduces problems into the code, which it can with flixel. So for right now:
      1. modify the `Instrumentation.hx` handling of returns in the `replaceReturn` function like this:
      ```
      	return relocateExpr(macro {
            var result = ${instrumentExpr(expr)};
            instrument.profiler.Profiler.exitFunction(__profiler__id__);
            // return cast result; REMOVE THE cast on this line
            return result;
         }, expr.pos);
      ```
      2. Unfortunately this will cause issues in other places. These can be worked around by skipping instrumenting of certain classes. Here you have two choices. Either
         1. In your project.xml you will need to exclude the entire `flixel.graphics.frames` package.

         OR
         
         2. In your project.xml you will need to exclude the `flixel.graphics.frames.bmfont` package.
         3. In FlxFramesCollection.hx you will need add an @:ignoreInstrument metadata on the class:
         ```
         @:ignoreInstrument
         class FlxFramesCollection implements IFlxDestroyable
         {
  	      /**
         ```
        The second option is useful if you need to instrument other frame classes.
   3. You will likely need to add a termination of the profiler to your exit function. `instrument` does try to auto-detect the exit calls but that's a little tricky to find in a flixel app. So I added a hook in the exit handling of my game like this:
   ```
   	if (FlxG.keys.justReleased.ESCAPE)
		{
			instrument.profiler.Profiler.endProfiler();
			Sys.exit(0);
		}
   ```
   4. Modify your project.xml to include configuration for `instrument`:
   ```
   <haxelib name="instrument" />
	<haxeflag name="--macro"
		value='instrument.Instrumentation.profiling(["", "flixel"], ["source", "<ABSOLUTE_PATH_TO>/.haxelib/flixel/5,6,2"], ["flixel.animation", "flixel.effects", "flixel.input", "flixel.math", "flixel.path", "flixel.sound", "flixel.graphics.frames.bmfont", "flixel.text", "flixel.tile", "flixel.tweens", "flixel.ui", "flixel.util"])' />

	<!-- <haxeflag name="-D" value="profiler-console-hierarchy-reporter" /> -->
	<haxeflag name="-D" value="profiler-csv-reporter=sample2.csv" />

	<!--
    <haxeflag name="-D" value="dump=pretty" />
	<haxeflag name="-D" value="debug_instrumentation" />
    -->

   ```
   The `profiler-*` properties are explained in the haxe-instrument README. Choose whatever output you want to work with. I hope to write a new reporter to plug into Chrome's trace profiler.
   
   The third parameter to `instrument.Instrumentation.profiling(` is a list of packages to exclude from instrumenting. I have yet to fully refine this list but most of these packages are not critical to rendering performance, so I've excluded them in my tests for now.

   The `dump=pretty` is useful when you want to see the macro output from the compiler. This is a Haxe compiler flag not an `instrument` package flag. In all likelihood you want need it when using the profiler unless you need to debug it.
   
   The `debug_instrumentation` flag turns on `instrument`'s own debugging which is useful if compilation fails because of the instrumentation.


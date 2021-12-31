# canvas-capture
A small wrapper around [CCapture.js](https://github.com/spite/ccapture.js) and [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm) to record the canvas as an image (png/jpeg), video (mp4/webm), or gif – all from the browser!

Demo at: [apps.amandaghassaei.com/canvas-capture/demo/](https://apps.amandaghassaei.com/canvas-capture/demo/)  
Video export currently only works in Chrome (see [Caveats](#caveats) for more details about browser support and server header requirements).

- [Installation](#installation)
- [Use](#use)
- [Caveats](#caveats)
- [Additonal Notes](#additional-notes)
- [License](#license)
- [Development](#development)

This project doesn't expose *all* the features of either CCapture.js or ffmpeg.wasm, but it packages some of the most useful functionality in a convenient way to be installed via npm and run in the browser (I'm mostly using this in  projects built with webpack).  I've also added:

- type declarations for CanvasCapture API
- helper functions to bind recording and screen-shotting to hotkeys
- ability to export zipped png/jpeg frames with [JSZip](https://github.com/Stuk/jszip)
- an optional recording indicator (red dot) on screen to let you know when recording is happening
- other optional modal dialog features  

## Installation

To install this package run:

```npm install github:amandaghassaei/canvas-capture```

I haven't released this on npm because I'm waiting for some updates in CCapture.js that will make it a cleaner dependency to this repo ([see Issue](https://github.com/spite/ccapture.js/issues/78)), but for stability you can reference a specific commit like this:

```npm install github:amandaghassaei/canvas-capture#774ae42```

You can also try adding [dist/canvas-capture.js](dist/canvas-capture.js) or [dist/canvas-capture.min.js](dist/canvas-capture.min.js) to your project and it should be available as `window.CanvasCapture` or possibly globally as `CanvasCapture`.  I have not tested this yet, please let me know if this works!


## Use

There are a few ways to call CCapture. You can bind hotkeys to start/stop recording:

```js
import * as CanvasCapture from 'canvas-capture';

// Initialize and pass in canvas.
CanvasCapture.init(document.getElementById('my-canvas'));

// Bind key presses to begin/end recordings.
CanvasCapture.bindKeyToVideoRecord('v', {
  format: 'mp4', // Options are optional, more info below.
  name: 'myVideo',
  quality: 0.6,
});
CanvasCapture.bindKeyToGIFRecord('g');
// Download a series of frames as a zip.
CanvasCapture.bindKeyToPNGFramesRecord('f', {
  onProgress: (progress) => { // Options are optional, more info below.
    console.log(`Zipping... ${Math.round(progress * 100)}% complete.`);
  },
}); // Also try bindKeyToJPEGFramesRecord().

// These methods take a single snapshot.
CanvasCapture.bindKeyToPNGSnapshot('p'); 
CanvasCapture.bindKeyToJPEGSnapshot('j', {
  name: 'myJpeg', // Options are optional, more info below.
  quality: 0.8,
});

function loop() {
   requestAnimationFrame(loop);

  // Render something...

  // You need to do this only if you are recording a video, gif, or frames.
  if (CanvasCapture.isRecording()) CanvasCapture.recordFrame();
}

loop();
```

Alternatively, you can call `beginXXXRecord` and `takeXXXSnapshot` directly:

```js
import * as CanvasCapture from 'canvas-capture';

// Initialize and pass in canvas.
CanvasCapture.init(document.getElementById('my-canvas'));

CanvasCapture.beginGIFRecord({ fps: 10 }); // Options are optional, more info below.
.... // Draw something.
CanvasCapture.recordFrame();
.... // Draw something.
CanvasCapture.recordFrame();
CanvasCapture.stopRecord();

// Now you may start another recording.
CanvasCapture.beginVideoRecord({ format: 'mp4' });
CanvasCapture.recordFrame();
....
CanvasCapture.stopRecord();

// Also try beginJPEGFramesRecord(jpegOptions) and beginPNGFramesRecord(pngOptions)

// Or you can call `takeXXXSnapshot` to take a single snapshot.
// No need to call `recordFrame` or `stopRecord` for these methods.
CanvasCapture.takePNGSnapshot();
CanvasCapture.takeJPEGSnapshot({ dpi: 600 }, (blob, filename) => {
  // Instead of automatically downloading the file, you can pass an optional callback
  // as the second argument to takeJPEGSnapshot() and takePNGSnapshot()
});

```

Available options for each capture type - this can be passed in as an optional argument to `bindKeyToXXX`, `beginXXXRecord`, or `takeXXXSnapshot`:

```ts
videoOptions = {
  format: 'mp4' | 'webm', // Defaults to 'mp4'.
  name: string, // Defaults to 'Video_Capture'.
  fps: number, // The frames per second of the output video, defaults to 60.
  quality: number, // A number between 0 and 1, defaults to 1.
  onExportProgress: (progress: number) => void, // progress is a number between 0 and 1.
  onExportFinish: () => void, // Callback after save complete.
  // Options below for ffmpeg conversion to mp4, not needed for webm export.
  ffmpegOptions?: { [key: string]: string }, // Defaults to
  // { '-c:v': 'libx264', '-preset': 'slow', '-crf': '22', '-pix_fmt': 'yuv420p' }
  // Internally the ffmpeg conversion runs with additional flags to crop to an even
  // number of px dimensions ('-vf crop=trunc(iw/2)*2:trunc(ih/2)*2', required for mp4)
  // and export no audio channel ('-an').
}
gifOptions = {
  name: string, // Defaults to 'GIF_Capture'.
  fps: number, // The frames per second of the output gif, defaults to 60.
  quality: number, // A number between 0 and 1, defaults to 1.
  onExportProgress: (progress: number) => void, // progress is a number between 0 and 1.
  onExportFinish: () => void, // Callback after save complete.
}
pngOptions = {
  name: string, // Defaults to 'PNG_Capture'.
  dpi: number, // Defaults to screen resolution (72 dpi).
  // onExportProgress and onExportFinish gives zipping updates for recording PNG frames
  // (only used by bindKeyToPNGFrames() and beginPNGFramesRecord()):
  onExportProgress: (progress: number) => void, // progress is a number between 0 and 1.
  onExportFinish: () => void, // Callback after save complete.
}
jpegOptions = {
  name: string, // Defaults to 'JPEG_Capture'.
  quality: number, // A number between 0 and 1, defaults to 1.
  dpi: number, // Defaults to screen resolution (72 dpi).
  // onExportProgress and onExportFinish gives zipping updates for recording JPEG frames
  // (only used by bindKeyToJPEGFrames() and beginJPEGFramesRecord()):
  onExportProgress: (progress: number) => void, // progress is a number between 0 and 1.
  onExportFinish: () => void, // Callback after save complete.
}
```

Note that changing the dpi of png/jpeg does not change the amount of pixels captured, just the dimensions of the resulting image.  

You can initialize `CanvasCapture` with the following options:

```js
import * as CanvasCapture from 'canvas-capture';

CanvasCapture.init(document.getElementById('my-canvas'), {
  verbose: true, // Verbosity of console output, default is true,
  showRecDot: true, // Show a red dot on the screen during records, default is false.
  recDotCSS: { right: '0', top: '0', margin: '10px' }, // CSS overrides for dot, default is {}.
  showAlerts: true, // Show alert dialogs during export in case of errors, default is false.
  showDialogs: true, // Show informational dialogs during export, default is false.
  ffmpegCorePath: './node_modules/@ffmpeg/core/dist/ffmpeg-core.js', // Path to a copy of
  // ffmpeg-core to be loaded asynchronously.  ffmpeg-core has not been included in this
  // library by default because it is very large (~25MB) and is only needed for mp4 export.
  // By default, ffmpegCorePath is set to load remotely from
  // https://unpkg.com/@ffmpeg/core@0.10.0/dist/ffmpeg-core.js
  // If you would like to load locally, you can set ffmpegCorePath to load from node_modules
  // ('./node_modules/@ffmpeg/core/dist/ffmpeg-core.js') using a copy of @ffmpeg/core installed
  // via npm, or copy the files at https://unpkg.com/browse/@ffmpeg/core@0.10.0/dist/ ,
  // save them in your project, and set ffmpegCorePath accordingly.
});
```

The baseline CSS for the record dot places it in the upper right corner of the screen, any of these params can be overwritten via options.recDotCSS:
```js
background: "red",
width: "20px",
height: "20px",
"border-radius": "50%", // Make circle.
position: "absolute",
top: "0",
right: "0",
"z-index": "10",
margin: "20px",
```

Additionally, you can set the verbosity of the console output at any time by:

```js
CanvasCapture.setVerbose(false);
```

I've also included a helper function to show a simple modal dialog with a title and message:

```js
const options = {
  // Set the amount of time to wait before auto-closing dialog, or -1 to disable auto-close.
  autoCloseDelay: 7000, // Default is -1.
};
// title and message are strings, options are optional.
CanvasCapture.showDialog(title, message, options);
```

## Caveats

Video export currently only works in Chrome.  mp4 and webm video export should be possible in Firefox once [this Firefox bug is addressed](https://bugzilla.mozilla.org/show_bug.cgi?id=1559743).

This repo depends on [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm/) to export mp4 video, not all browsers are supported:

>Only browsers with SharedArrayBuffer support can use ffmpeg.wasm, you can check [HERE](https://caniuse.com/sharedarraybuffer) for the complete list.  

In order for mp4 export to work, you need to configure your (local or remote) server correctly:

>SharedArrayBuffer is only available to pages that are [cross-origin isolated](https://developer.chrome.com/blog/enabling-shared-array-buffer/#cross-origin-isolation). So you need to host your own server with `Cross-Origin-Embedder-Policy: require-corp` and `Cross-Origin-Opener-Policy: same-origin` headers to use ffmpeg.wasm.

I've included a script for initializing a local server with the correct Cross-Origin policy at [canvas-capture/server.js](https://github.com/amandaghassaei/canvas-capture/blob/main/server.js).  If you need to start up your own server for testing, try running the code below to boot up a server at `localhost:8080`:

```sh
node node_modules/canvas-capture/server.js
```

If you're building an application with [webpack-dev-server](https://webpack.js.org/configuration/dev-server/), you can add the appropriate headers to your `webpack.config.js`:

```js
module.exports = {
  ...
  devServer: {
    ...
    headers: {
      "Cross-Origin-Embedder-Policy": "require-corp",
      "Cross-Origin-Opener-Policy": "same-origin",
    }
  }
}
```

If you're hosting an application on [Github Pages](https://pages.github.com/), I recommend checking out [coi-serviceworker](https://github.com/gzuidhof/coi-serviceworker) to get the correct headers on your page.  I was able to get this to work for my [demo page](https://apps.amandaghassaei.com/canvas-capture/demo/).

Additionally, you can test for browser support with the following methods:

```js
CanvasCapture.browserSupportsWEBM(); // Returns true if the browser supports webm recording.

CanvasCapture.browserSupportsMP4(); // Returns true if the browser supports mp4 recording.

CanvasCapture.browserSupportsGIF(); // Returns true if the browser supports gif recording.
```

I'm not aware of any browser limitations for the image export options (obviously, the browser must [support canvas](https://caniuse.com/?search=canvas) as a bare minimum).

Another thing to be aware of: this library defaults to pulling a copy of ffmpeg.wasm remotely from [unpkg.com/@ffmpeg/core@0.10.0/dist/](https://unpkg.com/@ffmpeg/core@0.10.0/dist/), so it requires an internet connection to export mp4.  If you want to host your own copy of ffmpeg-core, you'll need to provide a path to `ffmpeg-core.js` with the `ffmpegCorePath` option in `CanvasCapture.init()`.  Be sure to also include `ffmpeg-core.wasm` and `ffmpeg-core.worker.js` in the same folder.


## Additional Notes

- webm export is ready to download immediately after all frames are captured.  mp4 videos are generated by recording as webm, then converting into mp4 in the browser using [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm).  Since mp4 export requires an additional conversion step after the frames have been captured, you may find that ffmpeg conversion to mp4 takes too long for very large videos.
- webm videos are significantly larger than mp4.
- png export preserves the alpha channel of canvas, and jpeg/gif exporters will draw alpha = 0 as black, but the video exporter creates nasty artifacts when handling transparent/semi-transparent regions of the canvas – best to avoid this.  
- You cannot record gif and video (or multiple gifs / multiple videos) at the same time.  This appears to be a limitation of CCapture.js?  You can capture screenshots or record png/jpeg frames as zip while recording a video/gif.
- `beginXXXRecord` methods return a `capture` object that can be passed to `CanvasCapture.recordFrame(capture)` or `CanvasCapture.stopRecord(capture)` to target a specific recording.  If `recordFrame` or `stopRecord` are called with no arguments, all active captures are affected.
- gif.js (a dependency of CCapture.js) has some performance limitations and takes a significant amount of time to process after all frames have been captured, be careful if capturing a lot of frames.  Exported gifs tend to be quite large and uncompressed, you might want to optimize them further (I like [ezgif](https://ezgif.com/maker) for this).  
- Recording png/jpeg frames is currently set to save a zip with no compression with [JSZip](https://github.com/Stuk/jszip).  Even so, the zipping process takes some time and you might be better off saving the frames individually with `takeXXXSnapshot()` if you have a lot of files to save.  

### Converting WEBM to Other Formats

[Not all browsers](https://caniuse.com/sharedarraybuffer) support mp4 export, and even if they do, you may decide to export webm anyway for performance reasons (I tend to do this, they are much faster to export).  Webm is a bit annoying as a format though – I've found that I can play webm videos with [VLC player](https://www.videolan.org/vlc/), but the framerate tends to be choppy, and very few websites/programs support them.  If you want to convert your webms to mp4 (or any other format) after you've already downloaded them, I recommend using [ffmpeg](https://ffmpeg.org/) from the terminal:

`
ffmpeg -i PATH/FILENAME.webm -vf "crop=trunc(iw/2)*2:trunc(ih/2)*2" -c:v libx264 -preset slow -crf 22 -pix_fmt yuv420p -an PATH/FILENAME.mp4
`

`-vf "crop=trunc(iw/2)*2:trunc(ih/2)*2"` crops the video so that its dimensions are even numbers (required for mp4)  

`-c:v libx264 -preset slow -crf 22` encodes as h.264 with better compression settings  

`-pix_fmt yuv420p` makes it compatible with the web browser  

`-an` creates a video with no audio  

For Mac users: I highly recommend checking out [MacOS Automator](https://support.apple.com/guide/automator/welcome/mac) and creating a Quick Action for these types of conversions.  I have some instructions for that [here](https://github.com/amandaghassaei/ffmpeg-scripts).  I have a Quick Action for "Convert to MP4" that invokes the above ffmpeg command on whatever file I've selected in the Finder – it's a big time saver!


## License

The code in this repo is licensed under an MIT license, but it depends on other codebases and proprietary video codecs with different licenses.  Please be aware of this and check this project's dependencies for more info, specifically:

>@ffmpeg/core contains WebAssembly code which is transpiled from original FFmpeg C code with minor modifications, but overall it still following the same licenses as FFmpeg and its external libraries (as each external libraries might have its own license).


## Development

Pull requests welcome!

Install development dependencies by running:

```npm install```

To build `src` to `dist` run and recompile `demo`:

```npm run build```

Please note there is some weirdness around importing CCapture with npm.  I'm currently grabbing CCapture from a branch at `github:amandaghassaei/ccapture.js#npm-fix`.  I'm not proud of the changes I had to make to get this to work ([see diff here](https://github.com/amandaghassaei/ccapture.js/commit/7ada41933411c4b1bcde4cdb09eef03758838bc7)), but it's fine for now.  Also, in order to get the constructor to work correctly, I had to call `window.CCapture()` rather than using the module import directly.  You'll also see I had to assign the default export from CCapture to an unused temp variable to get everything to work:

```js
import CCapture from 'ccapture.js';
const temp = CCapture; // This is an unused variable, but critically necessary.

....

const capturer = new window.CCapture({
  format: 'webm',
  name: 'WEBM_Capture',
  framerate: 60,
  quality: 63,
  verbose: VERBOSE,
});
// This didn't work:
// const capturer = new CCapture({
//   format: 'webm',
//   name: 'WEBM_Capture',
//   framerate: 60,
//   quality: 63,
//   verbose: VERBOSE,
// });
```

Hopefully this will all be fixed in the future, see notes here:

https://github.com/spite/ccapture.js/issues/78  


### Demo

This repo also includes a demo for testing, currently hosted at [apps.amandaghassaei.com/canvas-capture/demo/](https://apps.amandaghassaei.com/canvas-capture/demo/).  You can find the demo source code at [demo/src/index.ts](demo/src/index.ts).

To build the `demo` folder, run:

```npm run build-demo```

To run the demo locally, run:

```npm run start```

This will boot up a local server with the correct Cross-Origin policies to support ffmpeg.wasm (a dependency for exporting mp4 video).  Navigate to the following address in your browser:

```localhost:8080/demo/```
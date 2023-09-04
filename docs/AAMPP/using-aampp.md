---
sidebar_position: 2
title: Using AAMPP in a plugin
---

#  Using AAMPP in a plugin

Starting with Volumio 3 plugins are forbidden from directly changing the ALSA configuration in `/etc/asound.conf`. This is to resolve the many issues that occurred when multiple plugins wanted to make changes to the ALSA configuration, and to simplify the behaviour of sofware volume control.

Clearly some plugins will need to customise the Volumio ALSA configuration to provide their functionality. The way that this is achieved is through ALSA contribution snippets.  

## What is an ALSA Contribution Snippet

An ALSA contribution snippet is a piece of ALSA configuration provided by a plugin in a specially named file. It is designed to be easily assembled (along with other contribution snippets) into a complete ALSA configuration.

The simplest valid ALSA contribution snippet would be a file named `in.out.conf` with the following content:

```
pcm.in {
    type copy
    slave.pcm "out"
}
```

There are several important things to note about this snippet:
* The file name is of the form `<a>.<b>.conf`, where `<a>` and `<b>` are the "input" and "output" of the contribution snippet
* The snippet does not attempt to directly address the audio hardware - it only deals with the behaviour of the plugin (in this case an no-op transform)
* Snippets should expect that their input samples are linear, and should output linear samples.
* The snippet must be prepared to handle a variety of input sample rates, bit depths, and channel counts. In most cases this will not need special handling, however if it does then an ALSA `plug` can be used.

### Providing an ALSA Contribution Snippet

Plugins must opt-in to providing ALSA contributions. This is done by setting the `has_alsa_contribution` property to `true` in the `volumio_info` of the plugin's `package.json`.

```json
{
  "name": "my_plugin",
  "version": "1.0.0",
  "description": "My super cool plugin",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "Volumio",
  "license": "ISC",
  "volumio_info": {
    "plugin_type": "audio_interface",
    "boot_priority":4,
    "has_alsa_contribution":true
  }
}
```
In addition to the `package.json` the plugin must place its ALSA contribution snippet(s) in a well defined location

### Static ALSA Contribution Snippets

Static snippets are ones which do not rely on any configuration or runtime values, and so can be hardcoded as part of the plugin archive. Most plugins will be able to use static snippets.

Static snippets live inside the plugin in a folder called `asound`.

### Dynamic ALSA Contribution Snippets

In some cases it is necessary to customise an ALSA snippet based on configuration, or data which can be found only once the plugin is installed. This obviously prevents the snippet from being included as part of the plugin archive.

Dynamic Contributions should be generated in `onVolumioStart` and written to the `asound` directory of plugin configuration folder. For example:


```js
var asoundFolder = self.commandRouter.pluginManager.getConfigurationFile(self.context, 'asound');
var promise  = libQ.nfcall(fs.outputFile, asoundFolder + '/in.out.conf', content, 'utf8');
```

:::tip
Since the custom alsa configuration sits at `/data/configuration/audio_interface/your_plugin/asound/in.out.conf` to make sure that if the folder does not exist gets created, `outputFile` is to be preferred to `writeFile`.
For this reason, use the `fs-extra` instead of the `fs` module in your plugin.
:::

If you need to change this configuration dynamically, for example when the outputdevice changes, make sure you instantiate a function that watches for audio devices changes and triggers an update of the configuration.

This can be achieved by adding this snippet inside the `onVolumioStart` function:

```
this.commandRouter.sharedVars.registerCallback('alsa.outputdevice', this.onAudioDeviceChanged.bind(this));
```

This function will execute the function `onAudioDeviceChanged` everytime the audio output device is changed. This function can look like:

```js
plugin.prototype.onAudioDeviceChanged = function() {
	var self = this;

	self.logger.info('Audio Device Changed, rebuilding plugin Configuration');
  // The function rebuildAlsaContribution will rebuild the configuration and resolve the promise once done
	self.rebuildAlsaContribution().then(()=>{
    // signal volumio that the whole AAMPP Configuration has to be refreshed
		self.commandRouter.rebuildALSAConfiguration();
	})
};
```

### ALSA Contribution Snippet Ordering

In general plugins should avoid trying to pick a particular order for snippets to run. There are, however, some cases where it is preferable to supply an ordering hint to Volumio.

In these scenarios the name of the ALSA contribution can be adjusted to include a third numeric segment `<a>.<b>.<priority>.conf`. The priority is an integer, and defaults to zero if not specified. Larger (more positive) values are put closer to the audio source. Lower (more negative) values are put closer to the hardware.

### Debugging

Debugging can be enabled on the Alsa Pipeline by creating a file in /data/alsadebug and restarting Volumio (and deleting it when not necessary).

```
touch /data/alsadebug
```

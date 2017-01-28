
  {
    "title"  : "Build your own remote garage door service",
    "authors": ["Joel Berger"],
    "date"   : "2017-01-28T09:34:49",
    "tags"   : [],
    "draft"  : true,
    "image"  : "",
    "description" : "Use Perl, a Raspberry Pi, and some components to monitor and control your garage door wherever you are",
    "categories": "hardware"
  }

Nearly every time I drive away from my house I wonder if I closed my garage door.
I'm not sure why, but I do.
Something about that mundane trivial task, which I almost always do, slips my mind as soon as it is accomplished, only to be questioned a later.
For years I've been aware of this problem and I finally decided to tackle it head-on.

## Supplies

Although I bought these in fits and starts, and ended up with plenty of other components, this was my useful purchase list.

* [Raspberry Pi 3 with power supply](https://smile.amazon.com/gp/product/B01C6FFNY4)
* A microSD card
* [Infrared proximity sensor](https://smile.amazon.com/gp/product/B01LPU9LV2)
* [Mountable prototype board](https://smile.amazon.com/Adafruit-Perma-Proto-HAT-Mini-Kit/dp/B00RHF1ZVY)
* [Solderable terminal blocks](https://smile.amazon.com/gp/product/B00X3WN392)
* Old telephone wire (long)
* Solderable jumper wire
* P2N2222 transistor
* 10kΩ resistor
* A soldering iron and solder (or access to them)

There are several Pi kits that come with a power supply and an SD card but they usually came with lots of other stuff I didn't need.
The resistor, transistor and wire I had laying around so I don't have links; they are very common components though.
I also believe a PN2222 or other similar transistors should work, but they would be wired slightly differently.

Similarly the proximity detector is just the one I found; I'm sure others would work.
It came with no instructions so I found [this site](http://www.14core.com/wiring-the-e18-d80nk-infrared-distance-ranging-sensor/).
This one's part number (as seen in the link) is E18-D80NK, it seems like it is relatively easy to find online.
I like its tough construction, especially since it is the only part that is going to be near the door (and thus the elements).

Assuming you don't want to mount the Pi near an open portal to weather, it is going to need extension wiring.
This is where a long run of telephone cord came in handy.
The sensor has three wires so my usual fallback of speaker wire doesn't work.
Telephone cord has four conductors and should be fine for the low voltage/current that this sensor needs.

The prototype board is a handy thing.
It is a solderable variant of the standard breadboards that are often sold in kits with Arduinos and Pis.
It has a 40-pin header (that you solder on) which attaches directly to the Pi, sitting directly above it.
It is actually a standard form, called a Hardware Attached on Top (HAT) (read more about [HATs here](https://www.raspberrypi.org/blog/introducing-raspberry-pi-hats/)).
HATs even have a little bit of onboard memory which you can use to initialize the Pi if needed.
For this project we won't but it is nice to know that the ability is there.

## The proto-board HAT

<img alt="THe proto-board HAT" style="max-width: 500px" src="/images/build-your-own-remote-garage-door-service/hat.jpg">

The proto-board HAT is where all of our custom electronics attach to the Pi.
There are three basic sections, the 40-pin header connected to the Pi, the door trigger, and the infrared sensor.

The header is just tedious, solder all 40 pins in place.
Be sure not to solder-bridge between two connections!

The trigger is a two-wire block and a P2N2222 transistor (left components of the image).
The wire block connects to the emitter and collector of the transistor.
The emitter side connects to the negative terminal of the garage door (via phone or speaker wire) and also via a jumper to the ground rail.
The collector connects to the positive terminal of the garage door.
The base is then connected to one of the generic GPIO pins of the Pi (I used [BCM 6](http://pinout.xyz/pinout/pin31_gpio6)).

The sensor is a three-wire block and a resistor (right components of the image).
Two terminals are simply for power (5V) and ground and are connected to those rails by jumper wires.
The terminal that connects to the sensor readout connects to a 10kΩ resistor.
The other end of the resistor is connected to another of the generic GPIO pins (I used [BMC 16](http://pinout.xyz/pinout/pin36_gpio16)).

I kept both of these systems to one side of the board in case I ever decide to add other home-automation to it.
If this is too compact for your tastes, you can of couse spread out along the board.

Note that there are several numbering schemes for the GPIO pins.
I recommend using the BCM numbers as those are the only ones that work with the sysfs interface.
<http://pinout.xyz> can be a very handy resource at this stages, though do note that the proto-board HAT didn't have room for a BCM 26 connection so resist the urge to use it.
Lots of the other pins have defined uses, read about each before you choose them.

Once everything is soldered, simply push-fit the board onto the GPIO pins.

## The Software

### Setting up the Pi

First you need to setup your Pi.
On a regular computer, follow the directions to [install Raspbian](https://www.raspberrypi.org/downloads/raspbian/) to the microSD card.
Raspbian is a derivative of Debian for the Pi.
I used the Lite version as I didn't need or want the desktop environment.
You can also use something called NOOBS to help install it or buy an SD card with it preinstalled, but this wasn't very hard.

The Pi and Raspbian are both products of the UK and the initial settings reflect that.
To change things to my native settings I followed [this post](http://rohankapoor.com/2012/04/americanizing-the-raspberry-pi/) which though it claims to Americanize it, it really is a general localization guide.

Most Raspberry Pi functions are exposed via a C library called [wiringpi](http://wiringpi.com/).
Indeed the [Perl bindings](https://metacpan.org/pod/RPi::WiringPi) I found on the CPAN use it.
If I were doing more complex things I would use both of these.
Indeed you might still want to install at least the wiringpi library as it provides a handy [command line interface](http://wiringpi.com/the-gpio-utility/) and especially the `gpio readall` command which is great for quickly reading the entire board.
As it is however, I needed so little that the [kernel's gpio sysfs](https://www.kernel.org/doc/Documentation/gpio/sysfs.txt) functionality was enough.

### The user interface

I wanted a really simple UI.
Just a garage door that was visibly up or down.
Click the door to open or close.

Because I wanted real-time updates of door state, I didn't want to have to reload the page.
With a websocket connection to the backend, pushing state regularly (or even only on change if desired), this should easily be possible.
But it did mean that the image would have to be responsive rather than just two independent images for open or closed.

I decided to make a single SVG image that could be manipulated with javascript and CSS.
I am not very good at CSS and especially positioning elements.
I figured I could just do that in the SVG editor for the most part.

I used [Inkscape](https://inkscape.org) to draw an SVG image of a garage door and a car in a box.
If I had any talent at that, it would be better.
That said, you can [use mine](https://github.com/jberger/CarPark/blob/master/art/car.min.svg) and skip this all together if you want.

Each image is on a separate layer of the SVG.
Inksapce lets you modify the resulting XML directly, which I used to add my own id attributes to each layer.
When exporting, pick "optimized SVG" and it will give you options, be sure that it keeps your manually added ids and adds the `viewBox` attribute.
Then in the exported file, remove the width and height attributes; with the `viewBox`, the SVG is now much more responsive to browser size.

The SVG is then directly included in the HTML template (via Mojo's `include` helper).
When an SVG image is included directly into HTML, it is part of the usual DOM and can be interacted with as such.
When the websocket informs the browser that the door state has changed I can then use javascript to add an `open` class.

```javascript
var ws = new WebSocket('<%= url_for('socket')->to_abs %>');
ws.onmessage = function(e) {
  door = JSON.parse(e.data);
  var c = document.getElementById('layer-door').classList;
  if (door.open) {
    c.add('open');
  } else {
    c.remove('open');
  }
}
```

With as simple CSS transition, I can now slide the door layer up when it has the `open` class applied.

```css
#layer-door {
  transition: transform 3.0s ease;
}
#layer-door.open {
  transform: translateY(-100%);
}
@media (max-width: 800px) {
  svg { max-width: 100%; }
}
@media (min-width: 800px) {
  svg { max-height: 500px }
}
```

The image is included below.
Rather than being triggered by the door, I have the transition applied by the `hover` pseudo-class.
Hover over to see it move.

<style>
  svg {
    width: 150px;
  }
  svg #layer-door {
    transition: transform 3.0s ease;
  }
  svg #layer-door:hover {
    transform: translateY(-100%);
  }
</style>

<div style="text-align:center">
<!-- Created with Inkscape (http://www.inkscape.org/) -->
<svg id="svg2" xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#" style="enable-background:new" xmlns="http://www.w3.org/2000/svg" version="1.1" xmlns:cc="http://creativecommons.org/ns#" xmlns:dc="http://purl.org/dc/elements/1.1/" viewBox="0 0 300.60001 353.00041">
 <metadata id="metadata7">
  <rdf:RDF>
   <cc:Work rdf:about="">
    <dc:format>image/svg+xml</dc:format>
    <dc:type rdf:resource="http://purl.org/dc/dcmitype/StillImage"/>
    <dc:title/>
   </cc:Work>
  </rdf:RDF>
 </metadata>
 <g id="layer-car" transform="translate(0.3 -700.86)">
  <rect id="rect5836" ry="23.299" height="315" width="283" stroke="#000" y="724.66" x="8.5" stroke-width="0.6" fill="#ffcb48"/>
  <g id="g4324">
   <g id="g4319">
    <g id="g5873" transform="translate(363 8.1333)">
     <rect id="rect5871" ry="15.299" height="115" width="74" stroke="#000" y="858.36" x="-290" stroke-width="0.6"/>
    </g>
    <rect id="rect4300" ry="15.299" height="34" width="65" stroke="#fff" y="842.36" x="77.5" stroke-width="0.6"/>
   </g>
   <g id="g4315" stroke-width="0.6">
    <rect id="rect5879" ry="15.299" height="115" width="74" stroke="#000" y="866.5" x="153"/>
    <rect id="rect4304" ry="15.299" height="34" width="65" stroke="#fff" y="842.36" x="157.5"/>
   </g>
   <g stroke="#000" stroke-width="0.6">
    <path id="rect5821" d="m71.299 810.5c-12.908 0-23.299 10.39-23.299 23.29v115.4c0 12.908 10.391 23.299 23.299 23.299h161.4c12.91 0.01 23.3-10.39 23.3-23.29v-115.4c0-12.908-10.391-23.299-23.299-23.299h-161.4zm8 17h146.4c7.3678 0 13.299 5.931 13.299 13.299v29.402c0 7.3678-5.931 13.299-13.299 13.299h-146.4c-7.369 0-13.3-5.94-13.3-13.3v-29.402c0-7.3678 5.931-13.299 13.299-13.299z" fill="#808ac8"/>
    <rect id="rect5811" ry="8.25" height="85" width="37" y="934.5" x="26"/>
    <rect id="rect5815" ry="8.25" height="85" width="37" y="934.5" x="237"/>
   </g>
   <g fill="#808ac8">
    <rect id="rect4136" ry="23.299" height="85" width="53" y="916.5" x="19"/>
    <rect id="rect5817" ry="23.299" height="85" width="53" y="916.5" x="228"/>
    <rect id="rect5819" ry="23.299" height="96" width="230" stroke="#000" y="892.5" x="36.5" stroke-width="0.6"/>
   </g>
   <g stroke="#000" stroke-width="0.6">
    <rect id="rect5828" ry="7" height="18" width="33" y="911.5" x="52" fill="#ce3c3c"/>
    <rect id="rect5830" ry="7" height="18" width="33" y="911.5" x="218" fill="#ce3c3c"/>
    <rect id="rect5832" ry="7" height="34" width="82" y="938.5" x="111" fill="#fff"/>
   </g>
  </g>
 </g>
 <g id="layer-door" transform="translate(0.3 1.5002)" stroke-width="0.6">
  <g id="g6012" transform="translate(0 -1.5002)">
   <rect id="rect5977" ry="10.299" height="70" width="300" stroke="#333" y="0.3" x="0" fill="#bb8531"/>
   <g stroke="#9c6f27" fill="#ce973f">
    <rect id="rect6000" ry="10.299" height="42" width="60" y="14.3" x="18"/>
    <rect id="rect6006" ry="10.299" height="42" width="60" y="14.3" x="86"/>
    <rect id="rect6008" ry="10.299" height="42" width="60" y="14.3" x="154"/>
    <rect id="rect6010" ry="10.299" height="42" width="60" y="14.3" x="222"/>
   </g>
  </g>
  <g id="g6019" transform="translate(0 69.1)">
   <rect id="rect6021" ry="10.299" height="70" width="300" stroke="#333" y="0.3" x="0" fill="#bb8531"/>
   <g stroke="#9c6f27" fill="#ce973f">
    <rect id="rect6023" ry="10.299" height="42" width="60" y="14.3" x="18"/>
    <rect id="rect6025" ry="10.299" height="42" width="60" y="14.3" x="86"/>
    <rect id="rect6027" ry="10.299" height="42" width="60" y="14.3" x="154"/>
    <rect id="rect6029" ry="10.299" height="42" width="60" y="14.3" x="222"/>
   </g>
  </g>
  <g id="g6031" transform="translate(0,139.7)">
   <rect id="rect6033" ry="10.299" height="70" width="300" stroke="#333" y="0.3" x="0" fill="#bb8531"/>
   <g stroke="#9c6f27" fill="#ce973f">
    <rect id="rect6035" ry="10.299" height="42" width="60" y="14.3" x="18"/>
    <rect id="rect6037" ry="10.299" height="42" width="60" y="14.3" x="86"/>
    <rect id="rect6039" ry="10.299" height="42" width="60" y="14.3" x="154"/>
    <rect id="rect6041" ry="10.299" height="42" width="60" y="14.3" x="222"/>
   </g>
  </g>
  <g id="g6043" transform="translate(0 210.3)">
   <rect id="rect6045" ry="10.299" height="70" width="300" stroke="#333" y="0.3" x="0" fill="#bb8531"/>
   <g stroke="#9c6f27" fill="#ce973f">
    <rect id="rect6047" ry="10.299" height="42" width="60" y="14.3" x="18"/>
    <rect id="rect6049" ry="10.299" height="42" width="60" y="14.3" x="86"/>
    <rect id="rect6051" ry="10.299" height="42" width="60" y="14.3" x="154"/>
    <rect id="rect6053" ry="10.299" height="42" width="60" y="14.3" x="222"/>
   </g>
  </g>
  <g id="g6055" transform="translate(0 280.9)">
   <rect id="rect6057" ry="10.299" height="70" width="300" stroke="#333" y="0.3" x="0" fill="#bb8531"/>
   <g stroke="#9c6f27" fill="#ce973f">
    <rect id="rect6059" ry="10.299" height="42" width="60" y="14.3" x="18"/>
    <rect id="rect6061" ry="10.299" height="42" width="60" y="14.3" x="86"/>
    <rect id="rect6063" ry="10.299" height="42" width="60" y="14.3" x="154"/>
    <rect id="rect6065" ry="10.299" height="42" width="60" y="14.3" x="222"/>
   </g>
  </g>
 </g>
</svg>

<p><small>Hover for door motion effect</small></p>
</div>

### The webapp

Finally, I used Perl and [Mojolicious](http://mojolicio.us) to build the site and do the gpio interactions.
I used nginx as a reverse proxy to the Mojolicious app, mostly because I'm used to it.
It might be easier for this small project to simply start the webserver as root and [drop permissions](https://metacpan.org/pod/Mojolicious::Plugin::SetUserGroup).

Although it isn't quite ready for release to CPAN, you can see the current state of the project at <https://github.com/jberger/CarPark>.
The project is in a few layers.
The new [Mojo::File](http://mojolicio.us/perldoc/Mojo/File) was a nice complement to the gpio sysfs interface.
Basically I built a tiny model layer on top of that, and then wrapped a tiny API over the model.
The user-level interaction here are exposed as export/unexport methods and read-write accessors for pin's values and modes.
(Pins must be exported before they are available through the sysfs interface.)

On top of the gpio/sysfs models I build some higher level helpers.
These are more directly related to the door functionality, as evidenced by names like `is_door_open` and `toggle_door`.

Then there are the routes.
Of course there is authentication.
Once authenticated there are routes that give low level access to pins (for completeness) and high level actions.
There is also a websocket that (when connected) pushes the state of the door once per second.

### Configuration

Currently there are three settings (besides the [hypnotoad settings](http://mojolicious.org/perldoc/Mojolicious/Guides/Cookbook#Hypnotoad)) that are available via a configuration file.
The first is a hashref of pins to be used (which default to the ones I use, of course).
Then a hashref of users and passwords.
Note that at the moment, these are in claer text, this is will be corrected before a CPAN release I expect, but at the same time, if you have access to the passwords on the Pi you already have access to the Pi and can just open the the door that way.

Finally there is a hashref of plugins that are dynamically added to the app.
This is to allow the user to add additional functionality, especially say [Mojolicious::Plugin::ACME](https://metacpan.org/pod/Mojolicious::Plugin::ACME).
Which leads us to ...

### Security!

PLEASE PLEASE PLEASE secure your server/application before opening it to the world.
I can't emphasize this enough, this system is now a key to your house (or at least your garage).
[Let's Encrypt](https://letsencrypt.org/) is a free service that provides SSL certificates.
You REALLY want to use SSL with this program so that other people can't see your password as you login.
The aforementioned ACME plugin adds the ability for the application to negotiate for its own certificates via LE.
There are plenty of examples online for using SSL to secure your site using nginx (though Mojolicious can use SSL directly as well).


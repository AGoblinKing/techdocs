# Mithril as a Game Engine
I recently used Mithril as a Game Engine at the [8BitHistory](http://www.8bithistory.org/) Game Jam #1. We took Best Overall Game and you can find the source code + demo @ [https://github.com/SchoolHouseRock/SchoolHouseRock]. But enough about the game lets talk about the tech!

## Crash course on Mithril
[Mithril](http://lhorie.github.io/mithril/) is a MVC framework that generates a VDOM(Virtual DOM) and applies the diffs to the actual DOM. Why? Changing the DOM is really slow and you can cause render flapping by reading/writing/reading/etc. By queueing up changes into a requestAnimationFrame and then applying them all at once you can avoid the flap!

Okay, that sounds pretty sweet but what else?

The functional style of the VDOM generation within Mithril lends itself to easy data binding and the ability to drastically change the presentation with a simple property change.

Consider the following:

    var data = { foo : 1 };

    document.body.innerHTML = "<div> Foo is " + data.foo + "</div>";

Kinda disgusting but renders (Foo is 1) and has a huge inherent XSS flaw.

    var data = { foo : 1 },
        ele  = document.createElement("div");

    ele.innerHTML = "Foo is " + data.foo;

    document.body.appendChild(ele);

Less disgusting, still flawed. Kinda wordy.

    var data = { foo : 1 },
        m = require("mithril");

    m.module(document.body, {
        controller : function() {
            return data;
        },
        view : function(ctrl) {
            return m("div", "Foo is " + ctrl.foo);
        }
    });

Wow that was more wordy, but doesn't have the XSS flaw due to Mithril escaping tags by default. (m.trust("") to get around that)

----

Now lets try to change 1 to 2 after a few seconds.

    var data = { foo : 1 },
        ele  = document.createElement("div");

    ele.innerHTML = "Foo is " + data.foo;

    document.body.appendChild(ele);

    setTimeout(function() {
        data.foo = 2;
        ele.innerHTML = "Foo is " + data.foo;
    }, 4000);

Not terrible.

    var data = { foo : 1 },
        m = require("mithril");

    m.module(document.body, {
        controller : function() {
            return data;
        },
        view : function(ctrl) {
            return m("div", "Foo is " + ctrl.foo);
        }
    });

    setTimeout(function() {
        data.foo = 2;
        m.redraw(); // schedule a redraw
    }, 4000);

Very similar, a little nicer as we're only modifying the data instead of touching the element.

----

Now lets try to increase the number every time to click it.

    var data = { foo : 1 },
        ele  = document.createElement("div");

    (function renderEle() {
        ele.innerHTML = "Foo is " + data.foo;
    } ());

    ele.addEventListener("click", function() {
        data.foo++;
        renderEle();
    });

    document.body.appendChild(ele);

    setTimeout(function() {
        data.foo = 2;
        renderEle();
    }, 4000);

Got tired of repeating myself and abstracted the render of the Ele. Code is getting a little bit hard to parse.

    var data = { foo : 1 },
        m = require("mithril");

    m.module(document.body, {
        controller : function() {
            // could just use this as your controller
            this.data = data;

            this.handleClick = function(e) {
                this.data.foo++;
            };
        },
        view : function(ctrl) {
            return m("div", {
                onclick : ctrl.handleClick
            }, "Foo is " + ctrl.data.foo);
        }
    });

    setTimeout(function() {
        data.foo = 2;
        m.redraw(); // schedule a redraw
    }, 4000);

Changed up the controller creation. Note that I didn't have to m.redraw() inside of handleClick. Mithril redraws after an event handler from its view.


## Actual Game Code [Warning]
Now that you understand the basis of Mithril, lets look at some game code!

    var m = require("mithril"),
    r = require("./random");

    module.exports = function gridController(ctrl) {
        function makeArr(num) {
            var arr = [];
            for(var i = 0; i < num; i++) {
                var coord = makeCoord(i);
                arr.push({
                    x : coord.x,
                    y : coord.y,
                    n : i
                });
            }

            return arr;
        }

        function makeGrid(grid) {
            grid = grid || {};
            var opts = nearPlayer(grid) ? {
                class : "moveable",
                onclick : movePlayer.bind(null, grid)
            } : {};

            return m(".grid.grid-" + grid.type, opts, [
                grid.type && grid.n !== ctrl.loc ? m(".gridName", grid.type) : "",
                grid.n  === ctrl.loc ? m(".player") : ""
            ]);
        }

        ctrl.loc = 20;
        ctrl.grid = makeArr(25);

        ctrl.grid[1].type = "school";
        ctrl.grid[1].actions = [{
            name : "Enter",
            action : ctrl.go("school")
        }];

        ctrl.grid[9].type = "work";
        ctrl.grid[9].actions = [{
            name : "Enter",
            action : function() {
                ctrl.go("work")();
                ctrl.type("Another day...another dollar");
            }
        }];

        ctrl.grid[20].type = "home";
        ctrl.grid[20].actions = [{
            name : "Enter",
            action : ctrl.go("home")
        }];

        ctrl.grid[12].type = "store";
        ctrl.grid[12].actions = [{
            name : "Enter",
            action : ctrl.go("store")
        }];

        ctrl.grid[24].type = "bar";
        ctrl.grid[24].actions = [{
            name : "Enter",
            action : ctrl.go("bar")
        }, {
            name : "Pee on wall",
            action : function() {
                ctrl.type("You're gross...");
            }
        }];

        return function gridView(ctrl) {
            return m(".thegrid.vbox.flex", [
                m(".grid-row.hbox.flex", ctrl.grid.slice(0, 5).map(makeGrid)),
                m(".grid-row.hbox.flex", ctrl.grid.slice(5, 10).map(makeGrid)),
                m(".grid-row.hbox.flex", ctrl.grid.slice(10, 15).map(makeGrid)),
                m(".grid-row.hbox.flex", ctrl.grid.slice(15, 20).map(makeGrid)),
                m(".grid-row.hbox.flex", ctrl.grid.slice(20, 25).map(makeGrid))
            ]);
        };
    };

Okay, a little bit more complex... but is it? I setup some data within the gridController then render it within the gridView. It renders five rows and maps the array against makeGrid which returns a mithril element.

Here is the resulting HTML:

    <div class="thegrid vbox flex">
        <div class="grid-row hbox flex">
            <div class="grid grid-undefined"></div>

            <div class="grid grid-school">
                <div class="gridName">
                    school
                </div>
            </div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>
        </div>

        <div class="grid-row hbox flex">
            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-work">
                <div class="gridName">
                    work
                </div>
            </div>
        </div>

        <div class="grid-row hbox flex">
            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-store">
                <div class="gridName">
                    store
                </div>
            </div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>
        </div>

        <div class="grid-row hbox flex">
            <div class="grid grid-undefined moveable"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>
        </div>

        <div class="grid-row hbox flex">
            <div class="grid grid-home">
                <div class="player"></div>
            </div>

            <div class="grid grid-undefined moveable"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-undefined"></div>

            <div class="grid grid-bar">
                <div class="gridName">
                    bar
                </div>
            </div>
        </div>
    </div>

Any time the data is changed by you clicking on the grid it will automatically re-render the DOM and suddenly you have a new set of moveable tiles. This keeps your game logic up in the controller and out of the view. You tend to be blissfully unaware of your render layer as you work on your logic side.

## But where are my classes?

Fuck man, I don't care how you organize your game logic. You can create the prettiest logic side and then just pipe in the raw data to view... it doesn't matter. Thats the beauty of a system like this... your view doesn't care about how it got the data.

## How is debugging?

Wonderful, the only bugs I ran into were game logic issues. The view code tends to remain small and highly reusable. Consider the following menu code:

    var m = require("mithril");

    createjs.Sound.registerSound("assets/Audio/Sounds/ButtonPress.mp3", "ClickButton");
    createjs.Sound.registerSound("assets/Audio/Sounds/ButtonMouseOver.mp3", "ButtonMouseOver");

    function makeOpt(opt) {
        return m(".option.hbox", {
            onclick : function(){
                createjs.Sound.play("ClickButton");
                opt.action();
            },
            onmouseover : createjs.Sound.play.bind(null, "ButtonMouseOver")
        }, opt.name);
    };

    module.exports = function(opts, classes) {
        classes = classes || " ";
        return m(".menu", {
            class : classes
        }, opts.map(makeOpt));
    };

This was nearly the first thing I wrote for CollegeQuest and I reused it EVERYWHERE. All it does is take an Array and map it into a selectable list. When you mouseover it, play a sound... if you click it do the action. This turned out to be the core gameplay loop of the game... implemented in what? 21 lines? Check out some code using it.

    var m = require("mithril"),
    r = require("./random");

    createjs.Sound.registerSound("assets/Audio/Sounds/fall_down.wav", "fallDown");

    module.exports = function passedOutCtrl(ctrl) {
        ctrl.passedOut = {
            actions : [{
                name : "Get up!",
                action : function() {
                    ctrl.go("grid")();
                    if(r.clamp(0, 100) <= 25) {
                        ctrl.resources.money = 0;
                        ctrl.type("Your wallet was taken while you were unconcious.");
                        ctrl.unlocked.push("Robbed");
                    } else {
                        ctrl.type(r.one([
                            "You pick your sorry ass off the ground",
                            "Sleeping on concrete sucks."
                        ]));
                    }
                }
            }],
            init : function() {
                createjs.Sound.play("fallDown").setVolume(.5);
            }
        };

        return function passedOutView(ctrl) {
            return m(".flex.passed-out");
        };
    };

Show these actions when passed out and play a sound when you first pass out. Oh btw, please render this incredibly simple dom element(has the splash screen for being passed out).

This is how every single game location was implemented and they're all nearly this small.

## Conclusion
Definitely going to experiment with Mithril as a Game Engine more. I had a wonderful experience and got to make a polished game in a relatively short(48hr) amount of time. The SVG capabilities are interesting and I'm working on a THREE.js Mithril like wrapper.


Until Next Show,

- Josh







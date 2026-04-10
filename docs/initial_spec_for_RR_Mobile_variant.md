# Instructions for creating the mobile variant of the Docktor Layman Rescue Game 2026

## file name 

file is initial_spec_for_RR_Mobile_variant.md

## overview of specification

As you can examine and see, this older contains an HTML game that runs in a browser. It uses keyboard presses for commands—ASDW for left-right-down-up, which is typical, as well as keys like escape for pause, B for bomb, space bar to fire, F for pickup, and E for dropping soldiers, and 1 to 4 to deploy tanks, anti-aircraft fire, troops, and a demo van. Also, M is to fire a missile, etc. 

I would like to make this repo into a new game optimized for use in browsers on mobile devices without keyboards, so I would like to optimize for a large iPad in landscape mode and put a joystick control for ASDW (flying the helicopter), and buttons configured for Bomb (B), spacebar shoot (N), missiles (M), pause (P), and four buttons for the current 1-infantry, 2-tank, 3-AA, and 4 demo van. 

## Layout and UI design

The touch controls need to be configured near each other for use by one hand and the joystick control for, say, the index finger of the other hand.  Perhaps these go best on the top right and top left, with a way to swap sides or lefty/righty preference. 

We will also need to clean up the onscreen instructions and sync them with the actual buttons and commands. I don’t mean to dictate at all my idea for the layout, but to illustrate what I am thinking. You could plan this out yourself based on popular and successful games and UI design. 

I have dropped a PNG image of a doctored playing screen with the on-screen controls added. Please go into plan mode, examine the existing game, this specification, and the illustration provided in file RR_MOBILE_RESCUE_RANGER_ONSCREEN_CONTROLS_03.png, and then document your design decisions in an MD file in a docs folder in this repo, and move the image I provided and this specification there also. [I might put them there myself right now. Yes, i just did.]
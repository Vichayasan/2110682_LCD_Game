## Reference vs. Modifications
Based on the reference provided (Muhamd Magdy's single-player dinosaur/runner game), this project has undergone a massive architectural overhaul.
Original Source Features (Muhamd Magdy):

•	Genre: Single-player infinite runner (similar to the Chrome Dinosaur game).</br>
•	Mechanics: One player jumps over randomly generated obstacles moving from right to left.</br>
•	Input: Single button, typically read using standard polling in the loop().</br>
•	Graphics: Custom characters for a dinosaur/hero and simple block obstacles.</br>
## Your Modifications & Additions:
•	Genre Shift: Converted into a 2-player Player-vs-Player (PvP) shooting game.</br>
•	Multiplayer Architecture: Added a second hero entity, facing the opposite direction, situated on the opposite side of the screen.</br>
•	Interrupt-Driven Inputs: Replaced digitalRead() polling with Hardware Interrupts for four distinct buttons, ensuring zero input lag during gameplay.</br>
•	Combat System: Added projectile mechanics (Bullet structs) with active hit-detection logic.</br>
•	Pre-Game Lobby: Created a synchronization lobby requiring both players to "Ready Up" before the game loop starts.</br>
•	Flicker-Free Rendering: Implemented a dual-buffer system (terrainUpper and terrainLower strings) to write the screen all at once, eliminating the visual tearing common in basic LCD games.</br>


## Hardware Design
The hardware is designed around the specific pinout of the Arduino Mega2560, utilizing its extended interrupt capabilities.

•	Display: A standard 16x2 LCD using a 4-bit parallel interface (RS=12, E=11, D4=5, D5=6, D6=7, D7=8).</br>
•	Player 1 Inputs:
  -	Jump: Pin 2 (External Interrupt INT0)
  -	Fire: Pin 18 (External Interrupt INT5)

•	Player 2 Inputs:
  - Jump: Pin 3 (External Interrupt INT1)
  - Fire: Pin 19 (External Interrupt INT4)
    
•	Wiring: All buttons utilize the Arduino's internal pull-up resistors (INPUT_PULLUP), meaning the buttons should wire the pins directly to Ground. The interrupts trigger on the FALLING edge.

## Software Architecture & Operation
The game operates on a continuous loop driven by a state machine and a custom rendering engine.
Interrupts & Debouncing
Instead of polling button states, the code uses Interrupt Service Routines (ISRs) like onJump01() and onFire01().

•	When a button is pressed, the hardware immediately halts the main program to run the ISR.

•	Inside the ISR, a non-blocking millis() check ensures that at least 50 milliseconds (DEBOUNCE_MS) have passed since the last press. This debouncing prevents mechanical switch noise from registering as multiple rapid presses.

•	If valid, a volatile boolean flag (e.g., jumpPushed01) is set to true for the main loop to process.

## The Hero State Machine
Each player's character is controlled by a 14-step state machine defined by the HERO_POSITION_* macros.

•	Running (Lower & Upper): States 1-3 and 12-14 cycle the character through three custom animation frames (HERO1_RUN1, HERO1_RUN2, HERO1_RUN3) to simulate legs moving.

•	Jumping: States 4-11 handle the jump arc. When a player presses jump, the state is forced to HERO_POSITION_JUMP_1. As the loop ticks forward via the advanceHero() function, the character ascends to the top row, stays there briefly, and falls back down.

## Combat System
The combat system relies on the Bullet struct, which tracks the bullet's X-coordinate, Y-coordinate (row 0 or 1), travel direction (+1 for P1, -1 for P2), and active status.

•	Spawning: Pressing fire calls spawnBullet(). It checks which row the hero is currently on using getHeroRow() and spawns the bullet directly in front of them.

•	Movement: moveBullet() increments or decrements the bullet's X-position. If it travels off the 16-character screen, it despawns.

•	Collision: checkHit() compares the active bullet's coordinates with the opponent's current X-position and Y-row. If they match, the game instantly transitions to the gameOver() state.

## Graphics and Rendering Engine
To prevent the LCD from flickering, the code does not draw directly to the screen piece-by-piece.
1.	Buffer Clearing: It initializes two 16-character arrays (terrainUpper and terrainLower) with spaces.
2.	Entity Placement: The drawHeroes() function calculates where both heroes and any active bullets are, placing the corresponding custom byte characters (or the - bullet character) into the arrays.
3.	Drawing: It appends a null-terminator \0 to the end of the arrays to turn them into valid C-strings, sets the cursor to the start of each row, and prints the entire row at once.
Game Flow
1.	Lobby: The game starts with playing = false. It waits in a while loop until both shown01 and shown02 are true (triggered by both players pressing jump).
2.	Main Loop: Once playing, the loop() function updates hero animations, spawns/moves bullets, checks for collisions, and draws the frame.
3.	Pacing: A fixed delay(100) at the end of the loop governs the game's speed, creating a steady 10 Frames Per Second (FPS) refresh rate.

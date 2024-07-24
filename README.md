# Arduino Lunar Lander with OctaPentaVeega VGA Shield
### (The OctaPentaVeega Module information is in another repository: Search for OctaPentaVeega)
I was given the excellent OctaPentaVeega - Arduino VGA shield by Jari Tulilahti. For a long time it sat on my Arduino-related hardware pile, since I was kind of busy with other projects - like the dual hardware speech synthesis box. One of the synths was in my drawer for about 20 years before I got around to actually finishing it. One day I might even document it.

 After some initial fumbling and mucking around, I finally came up with an idea what to do with it - the 1-dimensional text-based Lunar Lander simulator. Yeah, some snazzy IoT home automation display project would probably have been way cooler, but I like retro-y things, so Lunar Lander game it is. World is filled with temperature displays, anyway.

Now, when I say "simulator", I mean that in the broadest sense of the word - my understanding of physics (and related mathematics) has faded a long time ago, but after some creative copypasting from other, similar codebases I got it working. At least it looks like a game. The lofty project goals were:

Ultra-minimalistic user interface. It consists of one (1) potentiometer, so that box is checked.
Somewhat consistent simulation (oh well.)
At least one UI element that looks like graphics (there is a bar which shows the current thrust of the lunar module).
Picture below shows the UI all its g(l)ory. Please disregard the debug measurements present on the display.

# LunarLander
![img4](https://github.com/user-attachments/assets/86aea407-31cd-4f84-8d8d-38e0af860f3a)

 

The related hardware schematic is below. Pot in my circuit is A10K (linear 10 kilo-ohm).

# Schematic
![img3](https://github.com/user-attachments/assets/84bed45a-f4c1-41e9-a3fc-f14a6fc05190)

OctaPentaVeega
OctaPentaVeega is a cute little board with three attiny85's. Since Jari is a true embedded systems expert (unlike yours truly), his card generates the VGA timings and signals by software. With unbelievably small memory footprint.

From Arduino-amateurs point of view, the board is quite simple. Board generates a 640x480 standard timing VGA picture. The picture consists of 16 rows and 32 columns (numbered from zero). The data is fed from Arduino to OctaPentaVeega through one pin, as a SoftwareSerial port,  9600 bits per second. The data itself is ASCII text - but some ANSI codes are supported (so you can, for example, change colors or move the cursor). There are also mechanisms for rudimentary scrolling (line or screen, but to one direction only).

There is a "graphics" mode which somewhat resembles teletext - so no graphics primitives (lines, circles or so on), just byte-mapped blocks.

Talking to OctaPentaVeega from Arduino
The simplest way to talk to OctaPentaVeega from Arduino goes something like this:

#include <SoftwareSerial.h>
SoftwareSerial vga(9, 8)
	
In setup section we open the new port called "vga" with 9600 bps

vga.begin(9600);
	
And finally, in the running code we just send data to the serial port with write(), print() or println().

Lunar Lander
The code for the Lunar Lander is below. Yes! Global variables absolutely evrrrrywhere! Also my BASIC history shows in more ways than one, none of them good. Awful all around, but it seems to work.

 I've tried to comment every relevant section.

The game is played with potentiometer - I used standard linear 10K pot, but I feel the game would be better with a long-throw pot slider. Standard pot does not have very precise control, something with wider range would be better.

The thrust is applied as it would be on the real LEM - I've been told that the engine throttle rate was 3 seconds from idle to maximum thrust. This is simulated in the program with thrustrate variable.

Once the LEM reaches moon surface (altitude is zero), the game ends. Desirable outcome is to land with velocity less than 2m/s. Program comments about better (and smoother) pilot performance, though. Game is reset when player moves the potentiometer to both extremes (min and max setting).

I would like to include some graphic representation about g-forces and altitude, but the relatively slow interface between Arduino and OctaPentaVeega may prevent this. Maybe in the next version.

# Code

#include <SoftwareSerial.h>


SoftwareSerial vga(9, 8);   // We are talking to OctaPentaVeega through pin 8.
// Pin 9 is unused as the shield does not respond.

// Some static stuff for ANSI codes
const byte ESC[2] = {0x1B, 0x5B};
const byte CLR[4] = {0x1B, 0x5B, 0x32, 0x4A};
// Some static stuff for drawing a box on the UI - codes are graphic characters.
const byte UPBOX[16] = {137, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 151};
const byte MIDBOX[16] = {138, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 138};
const byte LOWBOX[16] = {136, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 133, 135};

// Time constant for physics calculations.
// As the full loop through software is quite consistent (around 100ms), there is
// no need to run the software by interrupts. Which of course would be better,
// but since we spend most of the time outputting things to screen, I did not bother.

const float sec = 0.1;

// Pins for potentiometer and thrust button (the first version used only
// a switch to control the thrust.
const int potPin = 2;
const int  buttonPin = 3;

// Physics constants
const float drymass = 7763;           // mass of the LEM module
const float gravity = 1.62;           // moon gravity in m/s^2
const float thrustmax = 43904;        // max thrust of LEM in newtons
const float kgper = 3049.86;          // fuel divider for thrust (as in Newton / kilogram)
const float thrustrate = 1463;        // engine throttle rate (maximum added newtons per 10th of a second)
const float potScale = 43904 / 1021;  // scale for potentiometer (analog reads from 0 to 1023, some leeway)


int buttonState = 0;
int potState = 0;
int lastButtonState = 0;
float altitude;
float fuel;
float thrust;
float velocity;
float totalmass;
long looptime;            //for measuring the loop

void InitVar() {
  altitude = 1500; // starting altitude
  fuel = 1490;   // fuel
  thrust = 0;    // thrust is set to zero
  velocity = 20;  // velocity should be more, but hey.
  totalmass = drymass + fuel;
}


void setup() {

  vga.begin(9600);             // start the serial port for OctaPentaVeega
  pinMode(buttonPin, INPUT);   // define button thrust input pin (not used)
  vga.write (CLR, 4);          // clear the VGA screen
  SetNoWrap();                 // set nowrap on VGA
  InitScreen();                // draw the initial UI elements
  InitVar();                   // Initialize flight variables
}


void loop() {
  // Draw screen, update physics, check if we landed and looplooploop forever
  DrawScreen();
  looptime = millis();
  UpdateStats();
  if (altitude <= 0) {
    endGame();  // go to landed screen if we indeed have landed
  }
}

void DrawScreen() {
  char block[32]; // byte array for thrust meter
  float thrustper = thrust / 439.04; // scale thrust
  int thrustblock = thrustper / 3.1; // find the setting

  for (int f = 0; f < 31; f++) // draw the thrust meter as byte array
  {
    if (f < thrustblock) {
      block[f] = 131;
    } else {
      block[f] = 32;
    }
  }

  // go to 0,0 on the screen and quickly output everything
  vga.write (ESC, 2);
  vga.write ("H"); // ANSI set cursor without parameters, same as 0,0
  vga.write(13);
  vga.print (altitude);
  vga.print (" m  ");
  vga.write(13);
  vga.write(13);
  vga.write(13);
  vga.print (velocity);
  vga.print (" m/s  ");
  vga.write(13);
  vga.write(13);
  vga.write(13);
  vga.print (thrustper);
  vga.print (" %  ");
  vga.write(13);
  vga.write(block, 31);
  vga.write(13);
  vga.write(13);
  vga.print (fuel);
  vga.print (" kg  ");
}

void InitScreen() {
  // Draw initial screen content.
  vga.write(CLR, 4);
  SetColor(30, 43);
  SetCursor(0, 0);
  vga.print ("   Altitude  ");
  SetCursor(3, 0);
  vga.print ("   Velocity  ");
  SetCursor(6, 0);
  vga.print ("    Thrust   ");
  SetCursor(9, 0);
  vga.print ("     Fuel    ");
  SetColor(37, 40);
  SetCursor(15, 0);
  vga.print ("G: ");
  vga.print (gravity);
  vga.print (" TX: ");
  vga.print (thrustmax);
  vga.print (" TR: ");
  vga.print (thrustrate);
  SetCursor (0, 16);
  SetColor (34, 40);

  vga.write(UPBOX, 16); // Create a box on the UI
  for (int f = 1; f < 7; f++)
  {
    SetCursor (f, 16);
    vga.write(MIDBOX, 16);
  }
  SetCursor (7, 16);
  vga.write(LOWBOX, 16);

  SetCursor (1, 17);  // some text and stuff
  SetColor (35, 40);
  vga.print ("OctaPentaVeega");
  SetCursor (3, 17);
  SetColor (32, 40);
  vga.print ("     MOON     ");
  SetCursor (4, 17);
  vga.print ("    LANDER    ");
  SetCursor (6, 17);
  SetColor (33, 40);
  vga.print ("(c) plzi 2017 ");
  SetColor (37, 40);
}

void UpdateStats()    // the awful physics simulation.
{
  float newaccel = 0;
  float newthrust = 0;
  totalmass = drymass + fuel;
  potState = analogRead(potPin);

  if (potState > 5) // yes, some leeway for the lower end of the pot...
  {
    if (fuel > 0) {
      newthrust = potState * potScale;
      if (potState > 1020) {
        newthrust = thrustmax;
      }
      if (newthrust > thrust) {
        thrust = thrust + thrustrate;
      } else {
        thrust = newthrust;
      }

      fuel = fuel - ((thrust / kgper) * sec);
      if (fuel < 0) {
        fuel = 0;
      }
      newaccel = gravity - ((thrust / totalmass));
    }
    else {
      thrust = 0;
      newaccel = gravity;
    }
  }
  else
  {
    thrust = thrust / 2;
    if (thrust < thrustrate) {
      thrust = 0;
    }
    newaccel = gravity;

  }
  velocity = velocity + (newaccel * sec);
  altitude = altitude - (velocity * sec);
}

void endGame() { // Game over - for now

  vga.write(CLR, 4);
  String mess = "";
  bool landed = true;
  bool smax = false;
  bool smin = false;

  if (velocity < 0.5) {
    mess = "Ultra smooth"; // select some text if we landed a-ok
  }
  else if (velocity < 1) {
    mess = "Smooth";
  }
  else if (velocity < 2) {
    mess = "Good";
  }
  else {
    landed = false;
  }

  SetCursor (4, 5);
  if (landed) {
    vga.print (mess);
    vga.print (" landing!");
  }
  else
  {
    vga.print ("You crashed the LEM!");
  }
  vga.println();
  vga.print ("End velocity was ");
  vga.print (velocity);
  vga.print (" m/s");
  vga.println();
  vga.print ("Fuel in tanks ");
  vga.print (fuel);
  vga.print (" kilograms");
  vga.println();
  vga.println ("Move pot from end to end");
  vga.println ("to start again");

  // move pot from end to end to continue
  do
  {
    potState = analogRead(potPin);
    if (potState < 2) {
      smin = true;
    }
    if (potState > 1020) {
      smax = true;
    }
  } while ((!smax) | (!smin));

  vga.write(CLR, 4);
  InitVar();
  InitScreen();
}

void SetCursor (int x, int y) {
  vga.write (ESC, 2);
  vga.print (x);
  vga.write (";");
  vga.print (y);
  vga.write ("H");
}

void SetColor (int fore, int back) {
  vga.write (ESC, 2);
  vga.print (fore);
  vga.write (";");
  vga.print (back);
  vga.write ("m");
}

void SetNoWrap() {
  vga.write (ESC, 2);
  vga.write ("?7l");
}

 

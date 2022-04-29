# Flipper-Plugin
Tutorial on how to build a basic "Hello world" plugin for Flipper Zero.

This tutorial includes: 

- A step by step story on how to create the base of a custom flipper plugin. (There are many other ways of creating apps for flipper, this example is based on the existing snake app)
- Sourcecode for a custom flipper app!

The tutorial has been writter during development of flappybird for flipper. 

# Hello World
In this tutorial a simple hello world plugin is added. That renders text in flippers canvas, and deals with user input (like closing the application). 

## Downloading the firmware
1. Clone or download [flipperzero-firmware](https://github.com/flipperdevices/flipperzero-firmware). 
```bash 
git clone https://github.com/flipperdevices/flipperzero-firmware
```
2. Create a folder for the custom plugin in `flipperzero-firmware/applications/`. For the hello-world app, this will be: `hello_world`. 
```bash
mkdir flipperzero-firmware/applications/hello_world
```
3. Create a new source file in the newly created folder. The file name has to match the name of the folder. 

```bash
touch flipperzero-firmware/applications/hello_world/hello_world.c
```


## Plugin Main 
For flipper to activate the plugin, a main function for the plugin has to be added. Following the naming convention of existing flipper plugins, this needs to be: `hello_world_app`. 

- Create an `int32_t hello_world_app(void* p)` function that will function as the entry of the plguin. 


For the plugin to keep track of what actions have been executed, we create a messagequeue. 
- A by calling `osMessageQueueNew` we create a new `osMessageQueueId_t` that keeps track of events. 

The view_port is used to control the canvas (display) and userinput from the hardware. In order to use this, a `view_port` has to be allocated, and callbacks to their functions registered. (The callback functions will later be added to the code)
- `view_port_alloc()` will allocate a new `view_port`. 
- `draw` and `input` callbacks originating from the `view_port` can be registerd with the functions
    - `view_port_draw_callback_set`
    - `view_port_input_callback_set`
- Register the `view_port` to the `GUI`

```c
int32_t hello_world_app(void* p) { 
    osMessageQueueId_t event_queue = osMessageQueueNew(8, sizeof(GameEvent), NULL); 
 
    // Set system callbacks
    ViewPort* view_port = view_port_alloc(); 
    view_port_draw_callback_set(view_port, render_callback, NULL);
    view_port_input_callback_set(view_port, input_callback, event_queue);
 
    // Open GUI and register view_port
    Gui* gui = furi_record_open("gui"); 
    gui_add_view_port(gui, view_port, GuiLayerFullscreen); 

    ...
}
```

## Callbacks 
Flipper will let the plugin know once it is ready to deal with a new frame or once a button is pressed by the user. 

**input_callback:**
Signals the plugin once a button is pressed. The event is queued in the event_queue. In the main thread the queue read and handled. 

A refrence to the queue is passed during the setup of the application.

```c
static void input_callback(InputEvent* input_event, osMessageQueueId_t event_queue) {
    furi_assert(event_queue); 

    PluginEvent event = {.type = EventTypeKey, .input = *input_event};
    osMessageQueuePut(event_queue, &event, 0, osWaitForever);
}
``` 

**render_callback:**
Signals the plugin when flipper is ready to draw a new frame in the canvas. For the hello-world example this will be a simple frame around the outer edges. 

```c
static void render_callback(Canvas* const canvas, void* ctx) {
    canvas_draw_frame(canvas, 0, 0, 128, 64);
}
```



In order to do something with this signal, we need to store this even 






### Render 




In order to make the hello world application appair in 


## "Main" 

The first called function of the application needs to be defined in `applications\applications.c`



The application can be called on any function n


Create a new folder `.c` file 

- code
    - add .c file 
    - main() 
    - draw_callback
    - input_callback
    - main loop
    
- makefile 



## Register the plugin 
We have to tell flipper's build system that a new plugin is added. We can do this by adding references to 

The application needs to be registered in the menu to be called. This is possible by adding an entry to `applications\applications.c`. 

```c
#ifdef APP_FLAPPY_GAME
    {.app = flappy_game_app, .name = "Flipper Flappy Bird", .stack_size = 1024, .icon = &A_Plugins_14},
#endif
```



# Building the example

1. Clone or download [flipperzero-firmware](https://github.com/flipperdevices/flipperzero-firmware)
2. Copy the folder `hello_world/` into `flipperzero-firmware/applications/`
3. Add the application to flippers menu by:
    1. defining the function as a `extern int32_t`. 
        ```c
        // Plugins
        extern int32_t music_player_app(void* p);
        extern int32_t snake_game_app(void* p);
        extern int32_t hello_world(void* p);
        ``` 
    2. 



# How it works

## Application Folder
Create a folder with the application name `hello_world` in `applications`. In `applications\hello_world` add a new file `hello_world.c`. 
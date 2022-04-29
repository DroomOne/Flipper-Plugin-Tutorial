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
```sh 
git clone https://github.com/flipperdevices/flipperzero-firmware
```
2. Create a folder for the custom plugin in `flipperzero-firmware/applications/`. For the hello-world app, this will be: `hello_world`. 
```sh
mkdir flipperzero-firmware/applications/hello_world
```
3. Create a new source file in the newly created folder. The file name has to match the name of the folder. 

```sh
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
typedef struct {
    EventType type;
    InputEvent input;
} PluginEvent;

typedef enum {
    EventTypeTick,
    EventTypeKey,
} EventType;

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

## Main Loop and plugin State
The main loop runs during the lifetime of the plugin. For each loop we try to pop an event from the queue, and handle the queue item such as button input / plugin events. 

For this example we render a new frame, every time the loop is run. This can be done by calling `view_port_update(view_port);`

```c
    PluginEvent event; 
    for(bool processing = true; processing;) { 
        osStatus_t event_status = osMessageQueueGet(event_queue, &event, NULL, 100);

        if(event_status == osOK) {
            // press events
            if(event.type == EventTypeKey) {
                if(event.input.type == InputTypePress) {  
                    switch(event.input.key) {
                    case InputKeyUp:   
                    case InputKeyDown:   
                    case InputKeyRight:   
                    case InputKeyLeft:   
                    case InputKeyOk: 
                    case InputKeyBack: 
                        // Exit the plugin
                        processing = false;
                        break;
                    }
                }
            } 
        } else {
            FURI_LOG_D(TAG, "osMessageQueue: event timeout");
            // event timeout
        }

        view_port_update(view_port); 
    }
```

### Plugin State
Because of the callback system, the plugin is being manipulated by different threads. To overcome race conditions we have to create a shared object that is safe to use. 

1. Allocate a new PluginState struct, and initialise it before the main loop.
```c
typedef struct { 
} PluginState; 

// in main:
PluginState* plugin_state = malloc(sizeof(PluginState));
```
2. Using `ValueMutex` we create a mutex for the plugin state called `state_mutex`. 
3. Initalise the mutex for `PluginState` using `init_mutex()`
4. Pass the mutex as argument to `view_port_draw_callback_set()` so we can safely access the shared state from flippers thread. 

```c
typedef struct { 
} PluginState; 

int32_t hello_world_app(void* p) { 
    osMessageQueueId_t event_queue = osMessageQueueNew(8, sizeof(PluginEvent), NULL); 
    
    PluginState* plugin_state = malloc(sizeof(PluginState));
    ValueMutex state_mutex; 
    if (!init_mutex(&state_mutex, plugin_state, sizeof(PluginState))) {
        FURI_LOG_E(TAG, "cannot create mutex\r\n");
        free(game_state); 
        return 255;
    }

    // Set system callbacks
    ViewPort* view_port = view_port_alloc(); 
    view_port_draw_callback_set(view_port, render_callback, &state_mutex);
    view_port_input_callback_set(view_port, input_callback, event_queue);
...
```


### Main Loop 

Let's deal with the mutex in our main loop. So we can update values from the main loop based on user input. As an example, we will move a hello-world text through the screen. Based on user input. 

1. For this we add a `int x` and `int y` to your state. 
```c
typedef struct { 
    int x; 
    int y;
} PluginState; 
```

2. Initialise the values of the struct using a new `hello_world_state_init()` function. 
```c
static void hello_world_state_init(PluginState* const plugin_state) {
    plugin_state->x = 10; 
    plugin_state->y = 10
} 
```
3. Call it after allocating the object in the main function. 
```c
PluginState* plugin_state = malloc(sizeof(PluginState));
hello_world_state_init(plugin_state);
ValueMutex state_mutex; 
...
```

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
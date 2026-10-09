#include <furi.h>
#include <gui/gui.h>
#include <input/input.h>
#include <infrared_worker.h>
#include <infrared_transmit.h>
#include <string.h>

#define MAX_TIMINGS 1024
#define IR_FREQ     38000
#define IR_DUTY     0.33f

typedef enum {
    StateIdle,
    StateRecording,
    StateStopped,
} AppState;

typedef struct {
    Gui* gui;
    ViewPort* vp;
    FuriMessageQueue* queue;
    FuriMutex* mutex;
    InfraredWorker* worker;
    AppState state;
    uint32_t timings[MAX_TIMINGS];
    size_t count;
    bool has_signal;
    bool sent;
} IrApp;

static void rx_callback(void* ctx, InfraredWorkerSignal* signal) {
    IrApp* app = ctx;
    if(infrared_worker_signal_is_decoded(signal)) return;

    const uint32_t* t;
    size_t n;
    infrared_worker_get_raw_signal(signal, &t, &n);

    furi_mutex_acquire(app->mutex, FuriWaitForever);
    // Keep the first signal captured while recording
    if(!app->has_signal && n > 0 && n <= MAX_TIMINGS) {
        memcpy(app->timings, t, n * sizeof(uint32_t));
        app->count = n;
        app->has_signal = true;
    }
    furi_mutex_release(app->mutex);
    view_port_update(app->vp);
}

static void draw_callback(Canvas* canvas, void* ctx) {
    IrApp* app = ctx;
    furi_mutex_acquire(app->mutex, FuriWaitForever);

    canvas_clear(canvas);
    canvas_set_font(canvas, FontPrimary);
    canvas_draw_str(canvas, 2, 12, "IR Raw Recorder");
    canvas_set_font(canvas, FontSecondary);

    char buf[40];
    switch(app->state) {
    case StateIdle:
        canvas_draw_str(canvas, 2, 28, "Press OK to record");
        break;
    case StateRecording:
        canvas_draw_str(canvas, 2, 28, "Recording... point remote");
        canvas_draw_str(canvas, 2, 40, "at Flipper, press button");
        break;
    case StateStopped:
        snprintf(buf, sizeof(buf), "Captured: %u timings", (unsigned)app->count);
        canvas_draw_str(canvas, 2, 28, app->has_signal ? buf : "Nothing captured");
        if(app->sent) canvas_draw_str(canvas, 2, 40, "Sent!");
        break;
    }

    canvas_draw_str(canvas, 2, 62, "OK:rec/stop  Left:send");

    furi_mutex_release(app->mutex);
}

static void input_callback(InputEvent* event, void* ctx) {
    IrApp* app = ctx;
    furi_message_queue_put(app->queue, event, 0);
}

int32_t ir_raw_rec_app(void* p) {
    UNUSED(p);

    IrApp* app = malloc(sizeof(IrApp));
    memset(app, 0, sizeof(IrApp));
    app->mutex = furi_mutex_alloc(FuriMutexTypeNormal);
    app->queue = furi_message_queue_alloc(8, sizeof(InputEvent));
    app->worker = infrared_worker_alloc();
    app->state = StateIdle;

    infrared_worker_rx_enable_signal_decoding(app->worker, false);
    infrared_worker_rx_enable_blink_on_receiving(app->worker, true);
    infrared_worker_rx_set_received_signal_callback(app->worker, rx_callback, app);

    app->vp = view_port_alloc();
    view_port_draw_callback_set(app->vp, draw_callback, app);
    view_port_input_callback_set(app->vp, input_callback, app);
    app->gui = furi_record_open(RECORD_GUI);
    gui_add_view_port(app->gui, app->vp, GuiLayerFullscreen);

    InputEvent ev;
    bool running = true;
    while(running) {
        if(furi_message_queue_get(app->queue, &ev, FuriWaitForever) != FuriStatusOk) continue;
        if(ev.type != InputTypeShort) continue;

        switch(ev.key) {
        case InputKeyOk:
            if(app->state == StateRecording) {
                infrared_worker_rx_stop(app->worker);
                furi_mutex_acquire(app->mutex, FuriWaitForever);
                app->state = StateStopped;
                furi_mutex_release(app->mutex);
            } else {
                furi_mutex_acquire(app->mutex, FuriWaitForever);
                app->has_signal = false;
                app->count = 0;
                app->sent = false;
                app->state = StateRecording;
                furi_mutex_release(app->mutex);
                infrared_worker_rx_start(app->worker);
            }
            view_port_update(app->vp);
            break;

        case InputKeyLeft:
            if(app->state != StateRecording && app->has_signal) {
                infrared_send_raw_ext(app->timings, app->count, true, IR_FREQ, IR_DUTY);
                app->sent = true;
                view_port_update(app->vp);
            }
            break;

        case InputKeyBack:
            running = false;
            break;

        default:
            break;
        }
    }

    if(app->state == StateRecording) infrared_worker_rx_stop(app->worker);

    gui_remove_view_port(app->gui, app->vp);
    view_port_free(app->vp);
    furi_record_close(RECORD_GUI);
    infrared_worker_free(app->worker);
    furi_message_queue_free(app->queue);
    furi_mutex_free(app->mutex);
    free(app);
    return 0;
}

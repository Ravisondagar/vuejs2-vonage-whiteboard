<template>
    <div class="classroom-whiteboard">
        <canvas
            :id="canvasId"
            :class="[erasing ? 'eraser' : 'pencil']"
            v-on:mousedown="mouseDown"
            @onresize="resizeCanvas"
        />
            <button class="btn-whiteboard-color">
                <color-picker
                      v-model="color"
                      :position="{right: '40px', bottom: '0px'}"
                      @change="change"
                      @afterChange="afterChange">
                </color-picker>                
            </button>
        <div class="classroom-whiteboard-action">
            <!-- <button
                v-for="color in colors"
                class="btn-whiteboard-color"
                @click="changeColor(color)"
                :class="[color, color == activeColor ? 'active' : '']"
            >
                {{ color }}
            </button> -->
            <button @click="erase" class="btn-undo-redo btn-erase">
                Eraser
            </button>
            <a
                @click="capture"
                :href="img"
                download="whiteboard_capture.png"
                class="btn-undo-redo"
            >
                Capture
            </a>
            <button @click="undo" class="btn-undo-redo">Undo</button>
            <button @click="redo" class="btn-undo-redo">Redo</button>
            <button @click="clearCanvas" class="btn-undo-redo">clear</button>
        </div>
    </div>
</template>

<script>
const paper = require("paper");
import Vue from 'vue';
import {ColorPicker, ColorPanel} from 'one-colorpicker'
Vue.use(ColorPanel)
Vue.use(ColorPicker)
var drawHistory = [];
export default {
    name: 'Vue2VonageWhiteboard', // vue component name
    props:['session', 'height', 'width'],
    data: () => ({
        path: null,
        scope: null,
        canvasId: "myId",
        colors: ["black", "blue", "red", "green", "yellow", "purple", "darkred"],
        color: "black",
        erasing: false,
        img: null,
        pathStack: [],
        undoStack: [],
        redoStack: [],
        activeColor: 'black',
        strokeWidth: 3.5,
        client: {},
        drawHistory: [],
        WhiteBoardStatus: {
            connection_id: null
        },
    }),
    created() {
        drawHistory = this.drawHistory;
        const _this = this;
        var drawHistoryReceivedFrom = null;
        const session = this.session;
        var requestHistory = function() {
            session.signal({
                type: "otWhiteboard_request_history"
            });
        };
        if (session) {
            if (session.isConnected()) {
                    requestHistory();
            }
            session.on({
                // sessionConnected: function() {
                //     requestHistory();
                // },
                "signal:otWhiteboard_update": function(event) {
                    if (event.from.connectionId !== session.connection.connectionId) {
                        _this.draw(event.data);
                        // this.drawUpdates(JSON.parse(event.data));
                        // this.$emit('otWhiteboardUpdate');
                    }
                },
                "signal:otWhiteboard_undo": function(event) {
                    if (event.from.connectionId !== session.connection.connectionId) {
                        _this.undoWhiteBoard(event.data.uuid);
                        _this.$emit("otWhiteboardUpdate");
                    }
                },
                "signal:otWhiteboard_redo": function(event) {
                    if (event.from.connectionId !== session.connection.connectionId) {
                        _this.redoWhiteBoard(event.data.uuid);
                        _this.$emit("otWhiteboardUpdate");
                    }
                },
                "signal:otWhiteboard_history": function(event) {
                    // We will receive these from everyone in the room, only listen to the first
                    // person. Also the data is chunked together so we need all of that person's
                    if (!drawHistoryReceivedFrom || drawHistoryReceivedFrom === event.from.connectionId) {
                        drawHistoryReceivedFrom = event.from.connectionId;
                        _this.drawUpdates(JSON.parse(event.data));
                        _this.$emit("otWhiteboardUpdate");
                    }
                },
                "signal:otWhiteboard_clear": function(event) {
                    if (event.from.connectionId !== session.connection.connectionId) {
                        _this.scope.project.activeLayer.removeChildren();
                        _this.drawHistory = [];
                        _this.pathStack = [];
                        _this.undoStack = [];
                        _this.redoStack = [];
                    }
                },
                "signal:otWhiteboard_request_history": function(event) {
                    if (event.from.connectionId !== session.connection.connectionId) {
                        if (drawHistory.length > 0) {
                            _this.batchSignal('otWhiteboard_history', event.from);
                        }
                    }
                }
            });
        }
    },
    methods: {
        change(response) {
            console.log(response);
            this.erasing = false;
        },
        afterChange(response) {
            console.log(response);
        },
        clearCanvas() {
            this.scope.project.activeLayer.removeChildren();
            this.drawHistory = [];
            this.pathStack = [];
            this.undoStack = [];
            this.redoStack = [];
            var signal = {
                type: "otWhiteboard_clear",
            };
            this.session.signal(signal);
        },
        pathCreate(scope) {
            var _this = this;
            var uuid = Math.random().toString(36).substring(2);
            _this.client.uuid = uuid;

            if (this.erasing) {
                var path = new paper.Path({
                    strokeWidth: 10,
                    strokeCap: "round",
                    strokeJoin: "round",
                    strokeColor: "white",
                    uuid: uuid
                });

                // learned about this blend stuff from this issue on the paperjs repo:
                // https://github.com/paperjs/paper.js/issues/1313

                // move everything on the active layer into a group with 'source-out' blend
                var tmpGroup = new paper.Group({
                    children: this.scope.project.activeLayer.removeChildren(),
                    blendMode: "source-out",
                    insert: false
                });

                // combine the path and group in another group with a blend of 'source-over'
                var mask = new paper.Group({
                    children: [path, tmpGroup],
                    blendMode: "source-over"
                });
                _this.path = path;
                return path;
            } else {
                var path = new paper.Path({
                    strokeColor: _this.color,
                    strokeJoin: "round",
                    strokeWidth: _this.strokeWidth,
                    uuid: uuid
                });
                _this.path = path;
                return path;
            }
        },
        createTool(scope) {
            scope.activate();
            return new paper.Tool();
        },
        mouseDown() {
            // in order to access functions in nested tool
            let self = this;
            var canvas = document.getElementById(this.canvasId);
            // const { left, top } = canvas.getBoundingClientRect();
            // const scaleX = canvas.width / event.target.width;
            // const scaleY = canvas.height / event.target.height;
            // const offsetX = event.clientX - left;
            // const offsetY = event.clientY - top;
            // const x = offsetX * scaleX;
            // const y = offsetY * scaleY;
            // var mode = this.erasing ? "eraser" : "pen";
            // create drawing tool
            this.tool = this.createTool(this.scope);
            if (this.erasing) {
                this.tool.minDistance = 10;
            }
            this.tool.onMouseDown = event => {
                var canvas = document.getElementById(this.canvasId);
                var mode = this.erasing ? "eraser" : "pen";
                // init path
                self.pathCreate(self.scope);
                // add point to path
                self.path.add(event.point);
                var update = {
                    uuid: self.client.uuid,
                    height: canvas.clientHeight,
                    width: canvas.clientWidth,
                    fromX: event.point.x,
                    fromY: event.point.y,
                    mode: mode,
                    color: self.color,
                    event: 'start'
                }
                self.drawHistory.push(update);
                var signal = {
                    type: "otWhiteboard_update",
                    data: update
                };
                // if (toConnection) signal.to = toConnection;
                self.session.signal(signal);

            };
            this.tool.onMouseDrag = event => {
                self.path.add(event.point);
                var canvas = document.getElementById(this.canvasId);
                var update = {
                    uuid: self.client.uuid,
                    height: canvas.clientHeight,
                    width: canvas.clientWidth,
                    toX: event.point.x,
                    toY: event.point.y,
                    event: 'drag'
                };
                self.drawHistory.push(update);
                var signal = {
                    type: "otWhiteboard_update",
                    data: update
                };
                // if (toConnection) signal.to = toConnection;
                self.session.signal(signal);
            };
            this.tool.onMouseUp = event => {
                // line completed
                self.path.add(event.point);
                self.pathStack.push(self.path);
                self.redoStack.push(self.path.uuid);
                var update = {
                    uuid: self.client.uuid,
                    event: 'end'
                };
                self.drawHistory.push(update);
                var signal = {
                    type: "otWhiteboard_update",
                    data: update
                };
                // if (toConnection) signal.to = toConnection;
                this.session.signal(signal);
            };
        },
        changeColor(color) {
            this.color = color;
            this.erasing = false;
            this.activeColor = color;
        },
        erase() {
            this.erasing = true;
            this.activeColor = null;
        },
        capture() {
            // event.preventDefault();
            var canvas = document.getElementById(this.canvasId);
            this.img = canvas.toDataURL("image/png");
        },
        undo() {
            if (this.redoStack.length == 0) return;
            var uuid = this.redoStack.pop();
            this.undoWhiteBoard(uuid);
            var signal = {
                type: "otWhiteboard_undo",
                data: {uuid: uuid}
            };
            this.session.signal(signal);
        },
        undoWhiteBoard(uuid) {
            this.undoStack.push(uuid);
            this.pathStack.forEach(function(path) {
                if (path.uuid === uuid) {
                    path.visible = false;
                }
            });
            // this.drawHistory.forEach(function(update) {
            //   if (update.uuid === uuid) {
            //     update.visible = false;
            //   }
            // });
        },
        redo() {
            if (this.undoStack.length == 0) return;
            var uuid = this.undoStack.pop();
            this.redoWhiteBoard(uuid);
            var signal = {
                type: "otWhiteboard_redo",
                data: {uuid: uuid}
            };
            this.session.signal(signal);
        },
        redoWhiteBoard(uuid) {
            this.redoStack.push(uuid);
            this.pathStack.forEach(function(path) {
                if (path.uuid === uuid) {
                    path.visible = true;
                }
            });
            // this.drawHistory.forEach(function(update) {
            //   if (update.uuid === uuid) {
            //     update.visible = true;
            //   }
            // });
        },
        drawUpdates(updates){
            var _this = this;
            updates.forEach(function (update) {
                _this.draw(update);
            });
        },
        draw(update) {
            this.drawHistory.push(update);
            var _this = this;
            switch (update.event) {
              case "start":
                var canvas = document.getElementById(this.canvasId);
                const path = new paper.Path();
                path.selected = false;
                path.strokeColor = update.color;
                path.strokeWidth = _this.strokeWidth;
                path.strokeCap = 'round';
                path.strokeJoin = 'round';
                path.uuid = update.uuid;
                if (update.mode === "eraser") {
                  path.strokeWidth = 10;
                  path.strokeColor = "white";

                  var tmpGroup = new paper.Group({
                      children: this.scope.project.activeLayer.removeChildren(),
                      blendMode: "source-out",
                      insert: false
                  });

                  // combine the path and group in another group with a blend of 'source-over'
                  var mask = new paper.Group({
                      children: [path, tmpGroup],
                      blendMode: "source-over"
                  });
                }

                if (update.visible !== undefined) {
                  path.visible = update.visible;
                }
                _this.scope.activate();
                var x = (canvas.clientWidth * update.fromX) / update.width;
                var y = (canvas.clientHeight * update.fromY) / update.height;
                const start = new paper.Point(x, y);
                path.moveTo(start);
                _this.scope.view.draw();

                this.pathStack.push(path);
                break;
              case "drag":
                this.pathStack.forEach(pathItem => {
                  if (pathItem.uuid === update.uuid) {
                    var canvas = document.getElementById(this.canvasId);
                    var x = (canvas.clientWidth * update.toX) / update.width;
                    var y = (canvas.clientHeight * update.toY) / update.height;
                    _this.scope.activate();
                    pathItem.add(new paper.Point(x, y));
                    // paper.view.draw();
                    _this.scope.view.draw();
                  }
                });
                break;
              case "end":
                this.pathStack.forEach(pathItem => {
                  if (pathItem.uuid === update.uuid) {
                    _this.redoStack.push(pathItem.uuid);
                    pathItem.simplify();
                    _this.scope.view.draw();
                  }
                });
                break;
            }
        },
        batchSignal(type, toConnection) {
            if (this.drawHistory.length > 0) {
                // We send data in small chunks so that they fit in a signal
                // Each packet is maximum ~250 chars, we can fit 8192/250 ~= 32 updates per signal
                var dataCopy = this.drawHistory.slice();
                var signalError = function (err) {
                  if (err) {
                    TB.error(err);
                  }
                };
                while(dataCopy.length) {
                    var dataChunk = dataCopy.splice(0, Math.min(dataCopy.length, 32));
                    var signal = {
                        type: type,
                        data: JSON.stringify(dataChunk)
                    };
                    if (toConnection) signal.to = toConnection;
                    this.session.signal(signal, signalError);
                }                
            }
        },
        resizeCanvas(){
            var canvas = document.getElementById(this.canvasId);
            this.scope.view.viewSize = new paper.Size(canvas.clientWidth, canvas.clientHeight);
            this.scope.view.draw();
        }
    },
    mounted() {
        var canvas = document.getElementById(this.canvasId);
        window.addEventListener('resize', this.resizeCanvas, false);
        // canvas.width = "100%";
        // canvas.height = "100%";
        this.scope = new paper.PaperScope();
        this.scope.setup(this.canvasId);
        // canvas.style.width = this.width+"px";
        // canvas.style.height = this.height+"px";

        // console.log(this.scope.view);
        // Set paper.js view size
        // this.scope.view.viewSize = new paper.Size(canvas.width, canvas.height);
        this.scope.view.draw();
        // Draw the view now:
        // this.scope.view.draw();
        // console.log(canvas.clientHeight, canvas.clientWidth);
    },
    beforeDestroy: function () {
        this.path = null;
        this.canvasId = "myId";
        this.color = "black";
        this.erasing = false;
        this.img = null;
        this.pathStack = [];
        this.undoStack = [];
        this.redoStack = [];
        this.activeColor = 'black';
        this.client = {};
        this.drawHistory = [];
    },
    destroyed() {
        console.log('destroyed component');
    }
};
</script>
<style scoped>
    .classroom-whiteboard {
        padding-left: 0px;
        width: 100%;
    }
    .classroom-whiteboard {
        height: 100%;
        position: relative;
    }
    .btn-whiteboard-color {
        position: absolute;
        right: 5px;
        bottom: -52px;
        border-radius: 50%;
    }
    button {
        box-shadow: none;
        cursor: pointer;
        font-family: Roboto,-apple-system,BlinkMacSystemFont,Segoe UI,Helvetica Neue,Arial,sans-serif,Apple Color Emoji,Segoe UI Emoji,Segoe UI Symbol,Noto Color Emoji;
        outline: 0;
    }
    .classroom-whiteboard-action {
        position: absolute;
        left: 0px;
        bottom: -60px;
        height: 52px;
        width: 100%;
        overflow: auto;
        background: #3A3A3A;
        -webkit-border-radius: 6px;
        -moz-border-radius: 6px;
        border-radius: 6px;
        padding: 8px;
        display: flex;
        overflow: hidden;
    }
    .btn-whiteboard-color {
        position: absolute;
        right: 5px;
        bottom: -52px;
        border-radius: 50%;
    }
    button:not(:disabled) {
        cursor: pointer;
    }
    .one-colorpicker {
        position: relative;
        display: inline-block;
        vertical-align: top;
    }
    .color-block {
        width: 32px;
        height: 32px;
        cursor: pointer;
        position: relative;
    }
    .classroom-whiteboard .pencil {
        cursor: url(https://dashboard.matrixtutors.org/images/cursor.png),auto;
        width: 100%;
        height: 100%;
        border: 0;
        border-radius: 10px;
        display: block;
        margin: 0;
        background-color: white;
    }
</style>

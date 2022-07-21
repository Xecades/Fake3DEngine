<template>
    <div id="toolbar">
        <span>焦距</span><el-slider v-model="focal" :min=0.1 :max=20 :step=0.1 />
        <span>物体高度</span><el-slider v-model="height" :min=0.3 :max=5 :step=0.1 />
        <span>摄像机高度</span><el-slider v-model="camera_height" :min=0.3 :max=5 :step=0.1 />
        <span>鼠标灵敏度</span><el-slider v-model="rotate_sensitivity" :min=0.03 :max=1.5 :step=0.01 />
        <span>移动速度</span><el-slider v-model="camera_step" :min=0.03 :max=1.5 :step=0.01 />
        <p>WSAD 键移动，Q 键逆时针旋转，E 键顺时针旋转，左右屏幕均可点击.</p>
        <p>本项目未使用任何第三方 3D 引擎.</p>
        <p>&copy; <a id="xec" href="https://xecades.xyz/" target="_blank">Xecades</a> 2022</p>
    </div>
    <div id="canvas"></div>
</template>

<script>
import * as d3 from "d3";
import extend from "extend";

// y = 1000 / (0.7701017917x + 5.0794922683)

const grid = 33;

const cw = 15;
const ch = 15;
const spacing = 1;
const camera_radius = 0.25;

const view_radius = Math.sqrt(cw * cw + ch * ch); // 视角半径
const view_step = 5;

const s = d3.scaleLinear().domain([0, 1]).range([0, grid]);

const sin = (t) => Math.sin(t / 180 * Math.PI);
const cos = (t) => Math.cos(t / 180 * Math.PI);
const tan = (t) => sin(t) / cos(t);
const csc = (t) => 1 / sin(t);
const sec = (t) => 1 / cos(t);
const arctan = (d) => Math.atan(d) * 180 / Math.PI;

function intersect(line1, line2) {
    let del = (line1.y2 - line1.y1) * (line2.x2 - line2.x1) - (line1.x1 - line1.x2) * (line2.y1 - line2.y2);
    if (del == 0)
        return false;

    let x = ((line1.x2 - line1.x1) * (line2.x2 - line2.x1) * (line2.y1 - line1.y1) + (line1.y2 - line1.y1) * (line2.x2 - line2.x1) * line1.x1 - (line2.y2 - line2.y1) * (line1.x2 - line1.x1) * line2.x1) / del;
    let y = -((line1.y2 - line1.y1) * (line2.y2 - line2.y1) * (line2.x1 - line1.x1) + (line1.x2 - line1.x1) * (line2.y2 - line2.y1) * line1.y1 - (line2.x2 - line2.x1) * (line1.y2 - line1.y1) * line2.y1) / del;
    return { x, y };
}

function isIntersect(line1, line2) {
    if (!(
            Math.min(line1.x1, line1.x2) < Math.max(line2.x1, line2.x2) &&
            Math.min(line2.y1, line2.y2) < Math.max(line1.y1, line1.y2) &&
            Math.min(line2.x1, line2.x2) < Math.max(line1.x1, line1.x2) &&
            Math.min(line1.y1, line1.y2) < Math.max(line2.y1, line2.y2)))
        return false;

    let u, v, w, z;
    u = (line2.x1 - line1.x1) * (line1.y2 - line1.y1) - (line1.x2 - line1.x1) * (line2.y1 - line1.y1);
    v = (line2.x2 - line1.x1) * (line1.y2 - line1.y1) - (line1.x2 - line1.x1) * (line2.y2 - line1.y1);
    w = (line1.x1 - line2.x1) * (line2.y2 - line2.y1) - (line2.x2 - line2.x1) * (line1.y1 - line2.y1);
    z = (line1.x2 - line2.x1) * (line2.y2 - line2.y1) - (line2.x2 - line2.x1) * (line1.y2 - line2.y1);
    return u * v <= 0.00000001 && w * z <= 0.00000001;
}

// console.log(isIntersect({x1: 1, y1: 1, x2: 1, y2: 10}, {x1: 0, y1: 5, x2: 5, y2: 3}));

export default {
    name: "App",
    data() {
        return {
            // map: [
            //     [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
            //     [1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1],
            //     [1, 0, 1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1, 0, 1],
            //     [1, 0, 1, 1, 1, 1, 0, 0, 1, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 1, 0, 0, 1, 1, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 1, 0, 0, 1, 1, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1, 1, 1, 0, 1],
            //     [1, 0, 1, 1, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 0, 1, 1, 1],
            //     [1, 0, 0, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 0, 1],
            //     [1, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1],
            //     [1, 0, 1, 1, 1, 1, 0, 0, 1, 1, 1, 1, 1, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
            // ],
            // map: [
            //     [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
            //     [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
            // ],
            map: [
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0],
                [0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0],
                [0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
                [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
            ],
            canvas: null,
            left: null,
            right: null,
            camera: null,
            camx: 5.5,
            camy: 11.5,
            viewang: 270, // 视角旋转角度
            viewl: null,
            viewr: null,
            mesh: [],
            keystatus: {
                forward: false,
                backward: false,
                left: false,
                right: false,
                clockwise: false,
                counterclockwise: false,
            },
            height: 2,
            focal: 5,
            camera_height: 1.5,
            rotate_sensitivity: 0.4,
            camera_step: 0.15,
            view_angle: 114,
        };
    },
    watch: {
        camx() {
            this.paintCamera();
            this.paintView();
            this.processMesh();
            this.paintMesh();
            this.paintFocal();
            this.render();
        },

        camy() {
            this.paintCamera();
            this.paintView();
            this.processMesh();
            this.paintMesh();
            this.paintFocal();
            this.render();
        },

        viewang() {
            this.paintView();
            this.processMesh();
            this.paintFocal();
            this.render();
        },

        height() {
            this.calcMesh();
            this.processMesh();
            this.paintMesh();
            this.render();
        },

        focal() {
            this.paintFocal();
            this.calcMesh();
            this.processMesh();
            this.paintMesh();
            this.render();
        },

        camera_height() {
            this.calcMesh();
            this.processMesh();
            this.paintMesh();
            this.render();
        },
    },
    methods: {
        initMap() {
            this.map = Array.from(Array(ch), () => new Array(cw).fill(0));
        },

        paintLeft() {
            for (let i = 0; i < cw; i++) {
                for (let j = 0; j < ch; j++) {
                    let color = this.map[i][j] ? "rgba(0, 0, 255, .3)" : "transparent";
                    d3.select(`#rect-${i}-${j}`).attr("fill", color);
                }
            }
        },

        toggle(i, j) {
            this.map[i][j] = +!this.map[i][j];
            this.paintLeft();
            this.printMap();
            this.calcMesh();
            this.processMesh();
            this.paintMesh();
            this.render();
        },

        printMap() {
            let pt = "";
            for (let i = 0; i < cw; i++) {
                let line = [];
                for (let j = 0; j < ch; j++) line.push(this.map[i][j]);
                pt += "[" + line.toString() + "]\n";
            }
            console.log(pt);
        },

        paintCamera() {
            this.camera
                .attr("cx", s(this.camx))
                .attr("cy", s(this.camy));
        },

        walk(d) {
            let newx = this.camx + this.camera_step * cos(this.viewang + d);
            let newy = this.camy + this.camera_step * sin(this.viewang + d);
            if (this.iscollide(newx, newy)) return;
            this.camx = newx;
            this.camy = newy;
        },

        rotate(d) {
            this.viewang += d * view_step;
        },

        paintView(){
            this.viewl
                .attr("x1", s(this.camx))
                .attr("y1", s(this.camy))
                .attr("x2", s(this.camx + view_radius * cos(this.viewang - this.view_angle / 2)))
                .attr("y2", s(this.camy + view_radius * sin(this.viewang - this.view_angle / 2)));
            
            this.viewr
                .attr("x1", s(this.camx))
                .attr("y1", s(this.camy))
                .attr("x2", s(this.camx + view_radius * cos(this.viewang + this.view_angle / 2)))
                .attr("y2", s(this.camy + view_radius * sin(this.viewang + this.view_angle / 2)));
        },

        iscollide(x, y) {
            for (let i = 0; i < cw; i++) {
                for (let j = 0; j < ch; j++) {
                    if (this.map[i][j] == 1) {
                        if (i <= x && x <= i + 1 && j <= y && y <= j + 1) {
                            return true;
                        }
                    }
                }
            }
            return false;
        },

        calcMesh() {
            this.mesh = [];

            // 上面
            for (let j = 0; j < ch; j++) {
                let l = 0;
                let ind = false;
                for (let i = 0; i < cw; i++) {
                    if (this.map[i][j] == 1 && !this.map[i]?.[j - 1]) {
                        if (!ind) l = i;
                        ind = true;
                    } else {
                        if (ind) this.mesh.push([[l, j, this.height, 0], [i, j, this.height, 0], [i, j, 0, 0], [l, j, 0, 0]]);
                        ind = false;
                    }
                    if (ind && i == cw - 1) this.mesh.push([[l, j, this.height, 0], [i + 1, j, this.height, 0], [i + 1, j, 0, 0], [l, j, 0, 0]]);
                }
            }

            // 下面
            for (let j = 0; j < ch; j++) {
                let l = 0;
                let ind = false;
                for (let i = 0; i < cw; i++) {
                    if (this.map[i][j] == 1 && !this.map[i]?.[j + 1]) {
                        if (!ind) l = i;
                        ind = true;
                    } else {
                        if (ind) this.mesh.push([[l, j + 1, this.height, 0], [i, j + 1, this.height, 0], [i, j + 1, 0, 0], [l, j + 1, 0, 0]]);
                        ind = false;
                    }
                    if (ind && i == cw - 1) this.mesh.push([[l, j + 1, this.height, 0], [i + 1, j + 1, this.height, 0], [i + 1, j + 1, 0, 0], [l, j + 1, 0, 0]]);
                }
            }

            // 左面
            for (let i = 0; i < cw; i++) {
                let t = 0;
                let ind = false;
                for (let j = 0; j < ch; j++) {
                    if (this.map[i][j] == 1 && !this.map?.[i - 1]?.[j]) {
                        if (!ind) t = j;
                        ind = true;
                    } else {
                        if (ind) this.mesh.push([[i, t, this.height, 0], [i, j, this.height, 0], [i, j, 0, 0], [i, t, 0, 0]]);
                        ind = false;
                    }
                    if (ind && j == ch - 1) this.mesh.push([[i, t, this.height, 0], [i, j + 1, this.height, 0], [i, j + 1, 0, 0], [i, t, 0, 0]]);
                }
            }

            // 右面
            for (let i = 0; i < cw; i++) {
                let t = 0;
                let ind = false;
                for (let j = 0; j < ch; j++) {
                    if (this.map[i][j] == 1 && !this.map?.[i + 1]?.[j]) {
                        if (!ind) t = j;
                        ind = true;
                    } else {
                        if (ind) this.mesh.push([[i + 1, t, this.height, 0], [i + 1, j, this.height, 0], [i + 1, j, 0, 0], [i + 1, t, 0, 0]]);
                        ind = false;
                    }
                    if (ind && j == ch - 1) this.mesh.push([[i + 1, t, this.height, 0], [i + 1, j + 1, this.height, 0], [i + 1, j + 1, 0, 0], [i + 1, t, 0, 0]]);
                }
            }
        },

        processMesh() {
            for (let i = 0; i < this.mesh.length; i++)
                for (let j = 0; j < this.mesh[i].length; j++)
                    this.mesh[i][j][3] = this.depthOf(this.mesh[i][j]);

            const maxDepth = arr => arr.reduce((a, b) => a[3] > b[3] ? a[3] : b[3]);
            const minDepth = arr => arr.reduce((a, b) => a[3] < b[3] ? a[3] : b[3]);

            // 面最大深度排序
            this.mesh.sort((a, b) => maxDepth(b) - maxDepth(a));

            // 输出数据
            let meshp = extend(true, [], this.mesh);
            let print = "";
            for (let i = 0; i < meshp.length; i++) {
                let v = meshp[i];
                v.sort((a, b) => b[3] - a[3]);
                print += "{";
                for (let j = 0; j < v.length; j++) {
                    print += `{${j}, ${v[j][3]}}, `;
                }
                print += "}, ";
            }
            console.log(print);

            // 合格检测
            let iter = 1;
            for (iter = 1; iter < meshp.length; iter++) {
                let p = meshp[iter - 1];
                let v = meshp[iter];
                if (minDepth(p) < maxDepth(v)) break;
            }
            if (iter == meshp.length)
                console.log("合格");
        },

        paintMesh() {
            this.left.selectAll(".mesh").remove();

            for (let i = 0; i < this.mesh.length; i++) {
                let v = this.mesh[i];
                this.left
                    .append("line")
                    .attr("stroke", "pink")
                    .attr("stroke-width", "2px")
                    .attr("class", "mesh")
                    .attr("x1", s(v[0][0]))
                    .attr("y1", s(v[0][1]))
                    .attr("x2", s(v[2][0]))
                    .attr("y2", s(v[2][1]));
            }
        },

        paintFocal() {
            this.focalLine
                .attr("x1", s(this.camx + this.focal * sec(this.view_angle / 2) * cos(this.viewang + this.view_angle / 2)))
                .attr("y1", s(this.camy + this.focal * sec(this.view_angle / 2) * sin(this.viewang + this.view_angle / 2)))
                .attr("x2", s(this.camx + this.focal * sec(this.view_angle / 2) * cos(this.viewang - this.view_angle / 2)))
                .attr("y2", s(this.camy + this.focal * sec(this.view_angle / 2) * sin(this.viewang - this.view_angle / 2)));
        },

        render() {
            this.right.selectAll("*").remove();
            for (let i = 0; i < this.mesh.length; i++) {
                let v = this.mesh[i];
                let j = 0;
                let pts = "";

                for (j = 0; j < v.length; j++) {
                    let t = (v[j][1] - this.camy) / (v[j][0] - this.camx);
                    let h = v[j][3];
                    if (h < 0) break;
                    let x = cw / 2 - this.focal * tan(this.viewang - arctan(t));
                    let y = ch / 2 - this.focal * (v[j][2] - this.camera_height) / h;
                    pts += `${s(x)},${s(y)} `;
                }
                if (j != v.length) continue;
                this.right
                    .append("polygon")
                    .attr("points", pts)
                    // .attr("fill", "#b2b2ff")
                    // .attr("stroke", "rgba(0, 0, 255, .7)")
                    .attr("fill", "rgba(0, 0, 255, .1)")
                    .attr("stroke", "rgba(0, 0, 255, .4)")
            }
        },

        mousemove(e) {
            if (e.movementX > 0) {
                this.rotate(this.rotate_sensitivity);
            } else {
                this.rotate(-this.rotate_sensitivity);
            }
        },

        depthOf(v) {
            return (v[0] - this.camx) * cos(this.viewang) + (v[1] - this.camy) * sin(this.viewang);
        },
    },
    mounted() {
        // this.initMap();
        let that = this;

        // 画布
        this.canvas = d3
            .select("#canvas")
            .append("svg")
            .attr("width", s(cw * 2 + spacing))
            .attr("height", s(ch));

        // 边框
        this.canvas
            .append("rect")
            .attr("x", 0)
            .attr("y", 0)
            .attr("width", s(cw))
            .attr("height", s(ch))
            .attr("stroke", "steelblue")
            .attr("stroke-width", "2px")
            .attr("fill", "transparent")
            .attr("id", "border-left");

        this.canvas
            .append("rect")
            .attr("x", s(cw + spacing))
            .attr("y", 0)
            .attr("width", s(cw))
            .attr("height", s(ch))
            .attr("stroke", "steelblue")
            .attr("stroke-width", "2px")
            .attr("fill", "transparent")
            .attr("id", "border-right");

        // 分割
        this.left = this.canvas
            .append("svg")
            .attr("x", 0)
            .attr("y", 0)
            .attr("width", s(cw))
            .attr("height", s(ch))
            .attr("id", "svg-left")
            .attr("style", "cursor: pointer");

        this.right = this.canvas
            .append("svg")
            .attr("x", s(cw + spacing))
            .attr("y", 0)
            .attr("width", s(cw))
            .attr("height", s(ch))
            .attr("id", "svg-right");

        // 格子
        for (let i = 0; i < cw; i++) {
            for (let j = 0; j < ch; j++) {
                let that = this;
                this.left
                    .append("rect")
                    .attr("x", s(i))
                    .attr("y", s(j))
                    .attr("id", `rect-${i}-${j}`)
                    .attr("width", grid)
                    .attr("height", grid)
                    .attr("stroke", "steelblue")
                    .attr("stroke-opacity", "0.1")
                    .attr("fill", "transparent")
                    .on("click", function () {
                        that.toggle(i, j);
                    });
            }
        }

        // 渲染
        this.calcMesh();
        this.processMesh();
        this.paintMesh();
        this.render();

        // 视角边界
        this.viewl = this.left
            .append("line")
            .attr("stroke", "red");
        
        this.viewr = this.left
            .append("line")
            .attr("stroke", "blue");
        
        this.paintView();

        // 摄像机
        this.camera = this.left
            .append("circle")
            .attr("cx", s(this.camx))
            .attr("cy", s(this.camy))
            .attr("r", s(camera_radius))
            .attr("fill", "gold");

        // 渲染俯视图
        this.paintLeft();

        // 焦距
        this.focalLine = this.left
            .append("line")
            .attr("stroke", "gray");

        this.paintFocal();

        // 键盘操作
        document.addEventListener("keydown", function (e) {
            switch (e.keyCode) {
                case 87: that.keystatus.forward = true; break;
                case 83: that.keystatus.backward = true; break;
                case 65: that.keystatus.left = true; break;
                case 68: that.keystatus.right = true; break;
                case 81: that.keystatus.clockwise = true; break;
                case 69: that.keystatus.counterclockwise = true; break;
            }
        });
        document.addEventListener("keyup", function (e) {
            switch (e.keyCode) {
                case 87: that.keystatus.forward = false; break;
                case 83: that.keystatus.backward = false; break;
                case 65: that.keystatus.left = false; break;
                case 68: that.keystatus.right = false; break;
                case 81: that.keystatus.clockwise = false; break;
                case 69: that.keystatus.counterclockwise = false; break;
            }
        });
        this.move_sensitivity = 1;
        setInterval(() => {
            if (that.keystatus.forward) that.walk(0);
            if (that.keystatus.backward) that.walk(180);
            if (that.keystatus.left) that.walk(-90);
            if (that.keystatus.right) that.walk(90);
            if (that.keystatus.clockwise) that.rotate(-this.rotate_sensitivity);
            if (that.keystatus.counterclockwise) that.rotate(this.rotate_sensitivity);
        }, 25);

        // 鼠标操作
        (() => {
            let cvs = document.getElementById("border-right");
            let svg = document.getElementById("svg-right");
            let mv = (e) => { that.mousemove(e); };
            cvs.addEventListener("click", () => {
                cvs.requestPointerLock();
            });
            svg.addEventListener("click", () => {
                svg.requestPointerLock();
            });
            document.addEventListener("click", () => {
                if (document.pointerLockElement)
                    document.exitPointerLock();
            });
            document.addEventListener("pointerlockchange", () => {
                if (document.pointerLockElement)
                    document.addEventListener("mousemove", mv, false);
                else
                    document.removeEventListener("mousemove", mv, false);
            }, false);
        })();
    },
};
</script>

<style lang="stylus">
#canvas
    position: absolute
    left: 50%
    top: 50%
    transform: translate(-50%, -50%)

#toolbar
    color: #333
    position: fixed
    left: 0
    top: 0
    width: 300px
    padding: 1rem

#border-right, #svg-right
    cursor: pointer

*   
    user-select: none

#xec
    text-decoration: none
    color: #409EFF
    &:hover
        color: #337ecc
</style>

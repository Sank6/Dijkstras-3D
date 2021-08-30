<template>
  <Renderer
    ref="renderer"
    antialias
    :orbit-ctrl="{ enableDamping: true }"
    resize="window"
    shadow
  >
    <Camera
      ref="camera"
      :position="{ y: (options.spacing * options.size), z: -(options.spacing * options.size * 1.75) }"
      :far="10000"
    />
    <Scene ref="scene">
      <AmbientLight color="#ffffff" :intensity="0.5" />
      <PointLight color="#ffffff" :intensity="1" :position="{ y: options.spacing * options.size * 1.5 }" cast-shadow :shadow-map-size="{ width: options.spacing * options.size * 2.5, height: options.spacing * options.size * 2.5 }" :shadow-camera="{far: 10000}" />
      <RectAreaLight color="#42f5ef" :position="{ x: 0, y: options.spacing, z: (options.spacing * options.size) }" v-bind="rectLightsProps" />
      <RectAreaLight color="#42f5ef" :position="{ x: (options.spacing * options.size), y: options.spacing, z: 0 }" v-bind="rectLightsProps" />
      
      <Plane receive-shadow :width="options.spacing * options.size * 2.5" :height="options.spacing * options.size * 2.5" :rotation="{ x: -Math.PI / 2 }" :position="{ y: -options.spacing * options.size * 0.6 }">
        <StandardMaterial :props="{ roughness: 0.1, metalness: 0.9 }"  >
          <Texture :props="texturesProps" src="/textures/Metal_Panels_009_basecolor.jpg" />
          <Texture :props="texturesProps" src="/textures/Metal_Panels_009_normal.jpg" name="normalMap" />
          <Texture :props="texturesProps" src="/textures/Metal_Panels_009_roughness.jpg" name="roughnessMap" />
          <Texture :props="texturesProps" src="/textures/Metal_Panels_009_ambientOcclusion.jpg" name="aoMap" />
        </StandardMaterial>
      </Plane>

      <InstancedMesh ref="imesh" :count="max" cast-shadow>
        <SphereGeometry :radius="options.size" cast-shadow />
        <PhongMaterial />
      </InstancedMesh>
    </Scene>
    
    <EffectComposer>
      <RenderPass />
      <UnrealBloomPass :strength="0.03" />
      <FXAAPass />
    </EffectComposer>
  </Renderer>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import {
  Object3D,
  Color,
  Line,
  Vector3,
  LineBasicMaterial,
  BufferGeometry,
  Matrix4,
  RepeatWrapping,
} from "three";
import * as dat from "dat.gui";
import {
  Camera,
  PointLight,
  RectAreaLight,
  Renderer,
  RendererPublicInterface,
  Scene,
  InstancedMesh,
  SphereGeometry,
  PhongMaterial,
  AmbientLight,
  DodecahedronGeometry,
  Plane,
  StandardMaterial,
  Texture,
  EffectComposer,
  RenderPass,
  UnrealBloomPass,
  FXAAPass,
} from "troisjs";

export default defineComponent({
  components: {
    Camera,
    PointLight,
    RectAreaLight,
    Renderer,
    Scene,
    InstancedMesh,
    SphereGeometry,
    PhongMaterial,
    AmbientLight,
    DodecahedronGeometry,
    Plane,
    StandardMaterial,
    Texture,
    EffectComposer,
    RenderPass,
    UnrealBloomPass,
    FXAAPass,
  },
  setup() {
    let max = 500;
    let options = {
        size: 5,
        spacing: 150,
        max: parseInt(localStorage.getItem("max") as string) || max,
        maxDist: parseInt(localStorage.getItem("maxDist") as string) || 3,
        showNeighbourLines:
          JSON.parse(localStorage.getItem("showNeighbourLines") as string) !==
          false,
        showPathLines:
          JSON.parse(localStorage.getItem("showPathLines") as string) !== false,
        visitedNodeColor: localStorage.getItem("visitedNodeColor") || "#ffff00",
        unVisitedNodeColor:
          localStorage.getItem("unVisitedNodeColor") || "#ffffff",
        pathLineColor: localStorage.getItem("pathLineColor") || "#ffffff",
        neighbourLineColor:
          localStorage.getItem("neighbourLineColor") || "#1f1f1f",
        speed: parseInt(localStorage.getItem("speed") as string) || 6,
        loop: JSON.parse(localStorage.getItem("loop") as string) !== false,
    }
    return {
      coords: [] as number[][],
      started: false,
      imesh: null as typeof InstancedMesh.mesh | null,
      renderer: null as RendererPublicInterface | null,
      scene: null as typeof Scene.scene | null,
      minDistances: new Array(max).fill(Infinity),
      minPath: new Array(max).fill([]) as number[][],
      visitedNodes: [] as number[],
      lines: [] as any[],
      max,
      pauseTime: 100,
      options,
      rectLightsProps: {
        lookAt: {y: options.spacing, z: 0},
        intensity: 0.2,
        width: options.size*options.spacing,
        height: options.size*options.spacing,
        helper: true,
      },
      texturesProps: {
        repeat: { x: 2, y: 2 },
        wrapS: RepeatWrapping,
        wrapT: RepeatWrapping,
      },
    };
  },
  mounted() {
    this.imesh = (this.$refs.imesh as typeof InstancedMesh).mesh;
    this.renderer = this.$refs.renderer as RendererPublicInterface;
    this.renderer.renderer.setClearColor(new Color(0x050505));
    this.scene = this.$refs.scene as typeof Scene.scene;

    this.renderer.onBeforeRender(this.animate);
    this.setupOptions();
    this.start();
  },
  methods: {
    start() {
      const dummy = new Object3D();
      for (let i = 0; i < this.options.max; i++) {
        dummy.position.set(this.rand(), this.rand(), this.rand());
        this.coords.push(dummy.position.toArray());
        dummy.updateMatrix();
        this.imesh?.setMatrixAt(i, dummy.matrix);
        this.imesh?.setColorAt(i, new Color(this.options.unVisitedNodeColor));
      }

      let m = new Matrix4();
      m.set(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0);
      for (let i = this.options.max; i < this.max; i++) {
        this.imesh.setMatrixAt(i, m);
      }
      this.imesh?.setColorAt(0, new Color(1, 0, 0));
      this.imesh?.setColorAt(1, new Color(0, 1, 0));
      if (this.imesh) {
        this.imesh.instanceMatrix.needsUpdate = true;
        this.imesh.instanceColor.needsUpdate = true;
      }
    },
    setupOptions() {
      const gui = new dat.GUI();
      let options = gui.addFolder("Options");
      options
        .add(this.options, "max", 75, this.max, 1)
        .name("Nodes")
        .onChange(this.restart);
      options
        .add(this.options, "maxDist", 1, 10, 1)
        .name("Reach")
        .onChange(this.restart);
      options
        .add(this.options, "speed", 0, 10)
        .name("Speed")
        .onChange(this.updateSpeed);
      options.add(this.options, "loop").name("Loop").onChange(this.restart);
      options
        .add(this.options, "showPathLines")
        .name("Path Lines")
        .onChange(this.updateLines);
      options
        .add(this.options, "showNeighbourLines")
        .name("Neighbour Lines")
        .onChange(this.updateLines);
      let colours = gui.addFolder("Colours");
      colours
        .addColor(this.options, "unVisitedNodeColor")
        .name("Unvisited Node")
        .onChange(this.updateColours);
      colours
        .addColor(this.options, "visitedNodeColor")
        .name("Visited Node")
        .onChange(this.updateColours);
      colours
        .addColor(this.options, "pathLineColor")
        .name("Path Line")
        .onChange(this.updateColours);
      colours
        .addColor(this.options, "neighbourLineColor")
        .name("Neighbours Line")
        .onChange(this.updateColours);

      let controls = gui.addFolder("Controls");
      controls.add(this, "restart").name("Restart");
      controls.add(this, "reset").name("Reset Options");
      this.updateSpeed();
    },
    animate() {
      if (!this.started) {
        this.started = true;
        this.step();
      }
    },
    async step() {
      // dijkstra
      let solved = false;
      let node = 0;
      let path: number[] = [];
      let total_distance = 0;
      while (solved === false) {
        await this.pause();

        if (node === undefined) {
          solved = true;
          this.solved(path, total_distance, true);
          break;
        }

        this.clearPath();
        if (this.options.showPathLines) this.drawPath(path);

        const neighbors = this.getNeighbors(node);
        for (let key in neighbors) {
          let dist = total_distance + neighbors[key];
          if (this.minDistances[parseInt(key)] > dist) {
            this.minDistances[parseInt(key)] = dist;
            this.minPath[parseInt(key)] = [...path, node];
          }
          if (this.options.showNeighbourLines)
            this.joinPoints(node, parseInt(key));
        }

        if (Object.keys(neighbors).includes("1")) {
          path.push(node);
          path.push(1);
          solved = true;
          this.solved(path, total_distance + neighbors[1]);
          break;
        }

        this.visitedNodes.push(node);

        // find next node
        let min = Infinity;
        let min_index = 0;
        for (let node_ in this.minDistances) {
          if (
            !this.visitedNodes.includes(parseInt(node_)) &&
            this.minDistances[node_] < min
          ) {
            min = this.minDistances[node_];
            min_index = parseInt(node_);
          }
        }

        node = min_index;
        path = [...this.minPath[node]];
        total_distance = this.minDistances[node];
      }
    },
    getNeighbors(node: number) {
      type Neighbours = {
        [key: number]: number;
      };
      const neighbours: Neighbours = {};
      for (let i = 0; i < this.coords.length; i++) {
        const dist = this.distance(this.coords[node], this.coords[i]);
        if (
          i !== node &&
          dist <
            this.options.maxDist *
              0.1 *
              this.options.spacing *
              this.options.size
        )
          neighbours[i] = dist;
      }
      return neighbours;
    },
    distance(a: number[], b: number[]) {
      return Math.sqrt(
        (a[0] - b[0]) ** 2 + (a[1] - b[1]) ** 2 + (a[2] - b[2]) ** 2
      );
    },
    joinPoints(pointA: number, pointB: number) {
      let points = [];
      if (![0, 1].includes(pointA)) {
        this.imesh?.setColorAt(
          pointA,
          new Color(this.options.visitedNodeColor)
        );
      }
      this.imesh.instanceColor.needsUpdate = true;

      points.push(new Vector3(...this.coords[pointA]));
      points.push(new Vector3(...this.coords[pointB]));
      const line = new Line(
        (new BufferGeometry() as any).setFromPoints(points),
        new LineBasicMaterial({ color: this.options.neighbourLineColor })
      );
      this.lines.push(line);
      this.scene?.add(line);
    },
    clearPath() {
      this.lines.forEach((line) => {
        this.scene?.remove(line);
        line.geometry.dispose();
        line.material.dispose();
      });
      this.lines = [];
    },
    drawPath(path: number[]) {
      let points = [];
      for (let i = 0; i < path.length; i++) {
        points.push(new Vector3(...this.coords[path[i]]));
        if (![0, 1].includes(path[i])) {
          this.imesh?.setColorAt(
            path[i],
            new Color(this.options.visitedNodeColor)
          );
        }
      }
      this.imesh.instanceColor.needsUpdate = true;
      const line = new Line(
        (new BufferGeometry() as any).setFromPoints(points),
        new LineBasicMaterial({ color: this.options.pathLineColor })
      );
      this.lines.push(line);
      this.scene?.add(line);
    },
    async solved(
      path: number[],
      total_distance: number,
      no_solution: boolean = false
    ) {
      this.imesh?.setColorAt(0, new Color(1, 0, 0));
      this.imesh?.setColorAt(1, new Color(0, 1, 0));
      this.clearPath();
      if (!no_solution) this.drawPath(path);
      if (this.options.loop) {
        await this.pause(2000);
        this.restart();
      }
    },
    updateSpeed() {
      this.pauseTime = 220 - this.options.speed * 20;
      localStorage.setItem("speed", this.options.speed.toString());
    },
    updateColours() {
      for (let i = 0; i < this.options.max; i++) {
        this.imesh?.setColorAt(i, new Color(this.options.unVisitedNodeColor));
      }
      for (let i = 0; i < this.visitedNodes.length; i++) {
        this.imesh?.setColorAt(
          this.visitedNodes[i],
          new Color(this.options.visitedNodeColor)
        );
      }
      this.imesh?.setColorAt(0, new Color(1, 0, 0));
      this.imesh?.setColorAt(1, new Color(0, 1, 0));
      this.imesh.instanceColor.needsUpdate = true;

      localStorage.setItem(
        "unVisitedNodeColor",
        this.options.unVisitedNodeColor
      );
      localStorage.setItem("visitedNodeColor", this.options.visitedNodeColor);
      localStorage.setItem(
        "neighbourLineColor",
        this.options.neighbourLineColor
      );
      localStorage.setItem("pathLineColor", this.options.pathLineColor);
    },
    updateLines() {
      localStorage.setItem(
        "showNeighbourLines",
        this.options.showNeighbourLines.toString()
      );
      localStorage.setItem(
        "showPathLines",
        this.options.showPathLines.toString()
      );
    },
    restart() {
      this.clearPath();
      this.coords = [];
      this.started = false;
      this.minDistances = new Array(this.max).fill(Infinity);
      this.minPath = new Array(this.max).fill([]) as number[][];
      this.visitedNodes = [];
      this.start();
      localStorage.setItem("loop", this.options.loop.toString());
    },
    rand() {
      return parseFloat(
        (
          Math.random() * this.options.spacing * this.options.size -
          (this.options.spacing * this.options.size) / 2
        ).toFixed(5)
      );
    },
    async pause(time: number = NaN) {
      await new Promise((resolve) =>
        setTimeout(resolve, time || this.pauseTime)
      );
      return;
    },
    reset() {
      if (confirm("Are you sure you want to reset everything?")) {
        localStorage.clear();
        location.reload();
      }
    },
  },
});
</script>

<style>
body,
html {
  margin: 0;
}
canvas {
  display: block;
}
</style>

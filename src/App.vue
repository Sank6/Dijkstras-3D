<template>
  <Renderer
    ref="renderer"
    antialias
    :orbit-ctrl="{ enableDamping: true }"
    resize="window"
  >
    <Camera ref="camera" :position="{ z: 10 }" />
    <Scene ref="scene">
      <AmbientLight color="#ffffff" intensity="0.5" />
      <InstancedMesh ref="imesh" count="500">
        <SphereGeometry radius="0.05" />
        <PhongMaterial />
      </InstancedMesh>
    </Scene>

    <EffectComposer>
      <RenderPass />
      <UnrealBloomPass :strength="0.8" :exposure="1" />
    </EffectComposer>
  </Renderer>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { Object3D, Color, Line, Vector3, LineBasicMaterial, BufferGeometry } from "three";
import {
  Camera,
  PointLight,
  Renderer,
  RendererPublicInterface,
  Scene,
  InstancedMesh,
  SphereGeometry,
  PhongMaterial,
  AmbientLight,

} from "troisjs";

export default defineComponent({
  components: {
    Camera,
    PointLight,
    Renderer,
    Scene,
    InstancedMesh,
    SphereGeometry,
    PhongMaterial,
    AmbientLight,
  },
  setup() {
    return {
      coords: [] as number[][],
      started: false,
      imesh: null as typeof InstancedMesh.mesh | null,
      renderer: null as RendererPublicInterface | null,
      scene: null as typeof Scene.scene | null,
      max: 100,
      maxDist: 3,
      minDistances: new Array(100).fill(Infinity),
      minPath: new Array(100).fill([]),
      visitedNodes: [] as number[],
    };
  },
  mounted() {
    this.imesh = (this.$refs.imesh as typeof InstancedMesh).mesh;
    this.renderer = this.$refs.renderer as RendererPublicInterface;
    this.scene = this.$refs.scene as typeof Scene.scene;

    this.renderer.onBeforeRender(this.animate);
    this.start();
  },
  methods: {
    start() {
      const dummy = new Object3D();
      for (let i = 0; i < this.max; i++) {
        dummy.position.set(this.rand(), this.rand(), this.rand());
        this.coords.push(dummy.position.toArray());
        dummy.updateMatrix();
        this.imesh?.setMatrixAt(i, dummy.matrix);
        this.imesh?.setColorAt(i, new Color(1, 1, 1));
      }
      this.imesh?.setColorAt(0, new Color(1, 0, 0));
      this.imesh?.setColorAt(1, new Color(0, 1, 0));
      if (this.imesh) {
        this.imesh.instanceMatrix.needsUpdate = true;
        this.imesh.instanceColor.needsUpdate = true;
      }
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
      let path: string[] = [];
      let total_distance = 0;
      while (solved === false) {
        await this.pause();
        const neighbors = this.getNeighbors(node);
        console.log(neighbors, node);
        for (let key in neighbors) {
          let dist = total_distance + neighbors[key];
          if (this.minDistances[parseInt(key)] > dist) {
            this.minDistances[parseInt(key)] = dist;
            this.minPath[parseInt(key)] = [...path, String(node)];
          }
        }
        
        if (Object.keys(neighbors).includes("1")) {
          this.drawPath(node, 1)
          solved = true;
          this.solved(path, total_distance + neighbors["1"]);
          break;
        }

        this.visitedNodes.push(node);
        
        // find next node
        let min = Infinity;
        let min_index = 0;
        for (let node_ in this.minDistances) {
          if (!this.visitedNodes.includes(parseInt(node_)) && this.minDistances[node_] < min) {
            console.log("changed")
            min = this.minDistances[node_];
            min_index = parseInt(node_);
          }
        }

        node = min_index;
        path = this.minPath[node].concat([String(node)]);
        total_distance = this.minDistances[node];

        console.log(node, path, total_distance);
      }
    },
    getNeighbors(node: number) {
      type Neighbours = {
        [key: string]: number
      }
      const neighbours: Neighbours = {};
      for (let i in this.coords) {
        const dist = this.distance(this.coords[node], this.coords[i])
        if (parseInt(i) !== node && dist < this.maxDist) neighbours[i] = dist;
      }
      return neighbours;
    },
    distance(a: number[], b: number[]) {
      return Math.sqrt(
        (a[0] - b[0]) ** 2 + (a[1] - b[1]) ** 2 + (a[2] - b[2]) ** 2
      );
    },
    drawPath(pointA: number, pointB: number) {
      let points = [];
      if (![0, 1].includes(pointA)) {
        this.imesh?.setColorAt(
          pointA,
          new Color(1, 1, 0)
        );
      }
      this.imesh.instanceColor.needsUpdate = true;

      points.push(
        new Vector3(...this.coords[pointA])
      );
      points.push(
        new Vector3(...this.coords[pointB])
      );
      const line = new Line(
        (new BufferGeometry() as any).setFromPoints(points),
        new LineBasicMaterial({ color: 0xffffff })
      );
      this.scene?.add(line);
    },
    solved(path: string[], total_distance: number) {
      this.drawPath(parseInt(path[path.length - 1]), parseInt(path[path.length - 2]));
      this.imesh?.setColorAt(0, new Color(1, 0, 0));
      this.imesh?.setColorAt(1, new Color(0, 1, 0));
      for (let i = 0; i < path.length - 1; i++) {
        this.drawPath(parseInt(path[i]), parseInt(path[i + 1]));
      }
      //alert(`Solved in ${total_distance}.`);
    },
    rand() {
      return parseFloat((Math.random() * 10 - 5).toFixed(5));
    },
    async pause() {
      await new Promise((resolve) => setTimeout(resolve, 50));
      return;
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

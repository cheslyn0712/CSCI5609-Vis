<script lang="ts">
  import { onMount } from "svelte";
  import * as THREE from "three";
  import { Scroll } from "$lib";
  import { addGround, loadModels, onWindowResize } from "$lib/Helper-3D";
  import { base } from "$app/paths";
  import type { TMovie } from "../../types";

  type Props = {
    movies: TMovie[];
  };
  let { movies }: Props = $props();

  const FLOOR = -250;
  const BOX_HEIGHT = 1;
  const BOX_WIDTH = 60;
  const BOX_DEPTH = 60;
  const BAR_SPACING = 80;
  const GENRES = [
    { genre: "Drama", color: 0x4e79a7 },
    { genre: "Comedy", color: 0xf28e2c },
    { genre: "Romance", color: 0xe15759 },
    { genre: "Documentary", color: 0x76b7b2 },
    { genre: "Action", color: 0x8c6bd8 },
    { genre: "Horror", color: 0x2f6b3b },
  ] as const;

  let progress = $state(0);
  let currentYear = $state(1950);

  let container: HTMLDivElement;
  let scene: THREE.Scene;
  let camera: THREE.PerspectiveCamera;
  let renderer: THREE.WebGLRenderer;
  let mixer: THREE.AnimationMixer;
  const clock = new THREE.Clock();

  const morphs: Array<THREE.Mesh> = [];
  const bars: THREE.Mesh[] = [];
  const labelSprites: THREE.Sprite[] = [];
  const targetScales: number[] = GENRES.map(() => 1);
  const targetCamera = new THREE.Vector3(0, FLOOR + 400, 800);
  const cameraLookAt = new THREE.Vector3(0, FLOOR + 120, 0);

  function getGenreCounts(moviesIn: TMovie[], year: number): Array<{ genre: string; count: number }> {
    const y0 = year - 5;
    const y1 = year + 5;
    const counts = new Map<string, number>(GENRES.map((g) => [g.genre, 0]));
    for (const movie of moviesIn) {
      const y = movie.year.getFullYear();
      if (y < y0 || y > y1) continue;
      for (const g of movie.genres) {
        const genre = g.trim();
        if (counts.has(genre)) counts.set(genre, (counts.get(genre) ?? 0) + 1);
      }
    }
    return GENRES.map((g) => ({ genre: g.genre, count: counts.get(g.genre) ?? 0 }));
  }

  function createLabelSprite(text: string): THREE.Sprite {
    const canvas = document.createElement("canvas");
    canvas.width = 256;
    canvas.height = 96;
    const ctx = canvas.getContext("2d");
    if (ctx) {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "rgba(255,255,255,0.92)";
      ctx.fillRect(8, 8, canvas.width - 16, canvas.height - 16);
      ctx.strokeStyle = "rgba(30,40,65,0.35)";
      ctx.strokeRect(8, 8, canvas.width - 16, canvas.height - 16);
      ctx.fillStyle = "#1f2941";
      ctx.font = "bold 28px Arial";
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      ctx.fillText(text, canvas.width / 2, canvas.height / 2);
    }
    const tex = new THREE.CanvasTexture(canvas);
    tex.needsUpdate = true;
    const material = new THREE.SpriteMaterial({ map: tex, transparent: true });
    const sprite = new THREE.Sprite(material);
    sprite.scale.set(80, 30, 1);
    return sprite;
  }

  function setStageCamera(p: number) {
    if (p <= 25) {
      targetCamera.set(0, FLOOR + 400, 800);
    } else if (p <= 50) {
      targetCamera.set(0, FLOOR + 250, 500);
    } else if (p <= 75) {
      targetCamera.set(-200, FLOOR + 300, 450);
    } else {
      targetCamera.set(0, FLOOR + 600, 600);
    }
  }

  function initScene(width: number, height: number) {
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(width, height);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFShadowMap;
    container.appendChild(renderer.domElement);

    camera = new THREE.PerspectiveCamera(33, width / height, 10, 5000);
    camera.position.set(0, FLOOR + 400, 800);
    camera.lookAt(cameraLookAt);

    scene = new THREE.Scene();
    scene.background = new THREE.Color(0xa8d8f0);
    new THREE.TextureLoader().load(
      `${base}/3d/sky.jpg`,
      (texture) => {
        texture.repeat.set(0.8, 1);
        scene.background = texture;
      },
      undefined,
      () => {},
    );

    scene.add(new THREE.AmbientLight(0xffffff, 0.85));
    const light = new THREE.DirectionalLight(0xffffff, 2.2);
    light.position.set(520, 1200, 950);
    light.castShadow = true;
    Object.assign(light.shadow.camera, {
      top: 1400,
      bottom: -1400,
      left: -1400,
      right: 1400,
      near: 200,
      far: 3500,
    });
    light.shadow.bias = 0.0001;
    scene.add(light);

    addGround(scene, FLOOR, `${base}/3d/grasslight-big.jpg`);

    const barGeometry = new THREE.BoxGeometry(BOX_WIDTH, BOX_HEIGHT, BOX_DEPTH);
    const startX = -((GENRES.length - 1) * BAR_SPACING) / 2;
    for (let i = 0; i < GENRES.length; i++) {
      const g = GENRES[i];
      const bar = new THREE.Mesh(
        barGeometry,
        new THREE.MeshStandardMaterial({
          color: g.color,
          roughness: 0.45,
          metalness: 0.08,
        }),
      );
      bar.castShadow = true;
      bar.receiveShadow = true;
      bar.position.set(startX + i * BAR_SPACING, FLOOR + (BOX_HEIGHT * bar.scale.y) / 2, 0);
      bars.push(bar);
      scene.add(bar);

      const label = createLabelSprite(g.genre);
      label.position.set(bar.position.x, FLOOR + 44, BOX_DEPTH * 0.85);
      labelSprites.push(label);
      scene.add(label);
    }

    const models = [
      {
        path: `${base}/3d/Flamingo.glb`,
        speed: 0,
        duration: 1,
        x: 350,
        y: FLOOR + 280,
        z: 110,
        scale: 0.5,
      },
      {
        path: `${base}/3d/Flamingo.glb`,
        speed: 0,
        duration: 1,
        x: 300,
        y: FLOOR + 240,
        z: 70,
        scale: 0.45,
      },
    ];
    mixer = loadModels(models, scene, mixer, morphs);

    renderer.setAnimationLoop(() => {
      const delta = Math.min(0.08, clock.getDelta());
      if (mixer) mixer.update(delta);

      for (let i = 0; i < bars.length; i++) {
        const bar = bars[i];
        const nextScale = bar.scale.y + (targetScales[i] - bar.scale.y) * 0.15;
        bar.scale.y = nextScale;
        bar.position.y = FLOOR + (nextScale * BOX_HEIGHT) / 2;
        labelSprites[i].position.y = FLOOR + nextScale * BOX_HEIGHT + 20;
      }

      const flightX = -400 + (progress / 100) * 800;
      if (morphs[0]) {
        morphs[0].position.x += (flightX - morphs[0].position.x) * 0.08;
        morphs[0].position.y = FLOOR + 280;
        morphs[0].position.z = 110;
      }
      if (morphs[1]) {
        morphs[1].position.x += (flightX - 55 - morphs[1].position.x) * 0.08;
        morphs[1].position.y = FLOOR + 240;
        morphs[1].position.z = 70;
      }

      camera.position.lerp(targetCamera, 0.08);
      camera.lookAt(cameraLookAt);
      renderer.render(scene, camera);
    });

    const onResize = () => {
      onWindowResize(camera, renderer, container.clientWidth || width, 500);
    };
    window.addEventListener("resize", onResize);
    return () => {
      window.removeEventListener("resize", onResize);
      renderer.setAnimationLoop(null);
      renderer.dispose();
    };
  }

  onMount(() => {
    const cleanup = initScene(container.clientWidth || 820, 500);
    return cleanup;
  });

  $effect(() => {
    currentYear = Math.round(1950 + (progress / 100) * (2020 - 1950));
    const counts = getGenreCounts(movies, currentYear);
    for (let i = 0; i < counts.length; i++) {
      targetScales[i] = Math.max(1, counts[i].count * 4);
    }
    setStageCamera(progress);
  });
</script>

<h2>3D Scrolly: Genre Flight</h2>
<p class="intro">The flock carries us through changing genre landscapes from 1950 to 2020.</p>

<Scroll bind:progress --scrolly-story-width="0.95fr" --scrolly-viz-width="1.2fr">
  <section class="story-step">
    <h3>Take Flight</h3>
    <p>We begin far away and fly into the scene. The flock introduces an 80-year story of summer movies.</p>
    <p>As the camera pushes in, the timeline starts moving year by year.</p>
  </section>

  <section class="story-step">
    <h3>1950s: The Golden Age of Drama</h3>
    <p>Post-war storytelling gave drama a strong presence in summer releases.</p>
    <p>The bars rise from the ground to reveal early genre balance.</p>
  </section>

  <section class="story-step">
    <h3>1980s: Comedy Takes Off</h3>
    <p>As years advance, comedy grows quickly and begins to challenge drama.</p>
    <p>The camera shifts to a side angle so relative heights are easier to compare.</p>
  </section>

  <section class="story-step">
    <h3>2020s: A Diverse Sky</h3>
    <p>By the streaming era, the composition becomes more mixed across several genres.</p>
    <p>We end with a high overview of the full distribution landscape.</p>
  </section>

  <section slot="viz" class="viz">
    <div class="hud"><b>Current year:</b> {currentYear}</div>
    <div class="canvas-wrap" bind:this={container}></div>
    <div class="legend">
      {#each GENRES as g}
        <span><i style:background={"#" + g.color.toString(16).padStart(6, "0")}></i>{g.genre}</span>
      {/each}
    </div>
  </section>
</Scroll>

<style>
  .intro {
    margin-top: 0;
    color: #4a4f5b;
  }

  .story-step {
    min-height: 100vh;
    border-left: 3px solid #e9ebf3;
    padding: 1rem 0 2rem 1rem;
  }

  .story-step h3 {
    margin: 0 0 0.5rem;
    color: #1f2941;
  }

  .story-step p {
    margin: 0.25rem 0;
    line-height: 1.5;
    color: #3d4761;
  }

  .viz {
    background: #f9fbff;
    border: 1px solid #dce5f2;
    border-radius: 10px;
    padding: 0.8rem;
  }

  .hud {
    margin-bottom: 0.45rem;
    color: #223049;
  }

  .canvas-wrap {
    width: 100%;
    min-height: 500px;
    border-radius: 8px;
    overflow: hidden;
    border: 1px solid #d1dced;
    background: #f5f8fd;
  }

  .legend {
    display: flex;
    gap: 0.85rem;
    flex-wrap: wrap;
    margin-top: 0.5rem;
    color: #2f3b56;
    font-size: 0.82rem;
  }

  .legend span {
    display: inline-flex;
    align-items: center;
    gap: 0.35rem;
  }

  .legend i {
    width: 12px;
    height: 12px;
    border-radius: 2px;
    display: inline-block;
  }
</style>

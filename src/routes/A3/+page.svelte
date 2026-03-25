<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { FontLoader } from "three/addons/loaders/FontLoader.js";
    import type { Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";

    import { addGround, onWindowResize, loadModels } from "$lib/Helper-3D";

    import * as d3 from "d3";
    import { base } from "$app/paths";
    import type { TMovie } from "../../types";

    type YearStack = {
        year: number;
        comedy: number;
        romance: number;
        drama: number;
    };

    const LAYERS = [
        { key: "comedy" as const, label: "Comedy", color: 0xdcc045, swatch: "#dcc045" },
        { key: "romance" as const, label: "Romance", color: 0xc94c6a, swatch: "#c94c6a" },
        { key: "drama" as const, label: "Drama", color: 0x2a9d9a, swatch: "#2a9d9a" },
    ];

    const CHIP_GEOM: Record<"comedy" | "romance" | "drama", THREE.BufferGeometry> = {
        comedy: new THREE.SphereGeometry(1, 14, 12),
        romance: new THREE.ConeGeometry(1, 1.35, 12),
        drama: new THREE.OctahedronGeometry(1, 0),
    };

    let container: HTMLElement;
    let camera: THREE.PerspectiveCamera;
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    const FLOOR = -250;
    const morphs: THREE.Mesh[] = [];
    let mixer: THREE.AnimationMixer;
    const clock = new THREE.Clock();

    let loadError = $state<string | null>(null);

    let framingStacks: YearStack[] = [];

    function hasGenre(m: TMovie, name: string): boolean {
        const n = name.toLowerCase();
        return m.genres.some((g) => g && g.toLowerCase() === n);
    }

    function buildYearStacks(movies: TMovie[]): YearStack[] {
        const valid = movies.filter((m) => m.year.getFullYear() > 1900);
        const byYear = d3.group(valid, (m) => m.year.getFullYear());
        const rows: YearStack[] = [];
        for (const [year, list] of byYear) {
            let comedy = 0;
            let romance = 0;
            let drama = 0;
            for (const m of list) {
                if (hasGenre(m, "Comedy")) comedy++;
                if (hasGenre(m, "Romance")) romance++;
                if (hasGenre(m, "Drama")) drama++;
            }
            rows.push({ year, comedy, romance, drama });
        }
        rows.sort((a, b) => a.year - b.year);
        return rows;
    }

    async function loadMovies(): Promise<TMovie[]> {
        const url = `${base}/summer_movies.csv`;
        const res = await fetch(url);
        if (!res.ok) {
            loadError = `Could not load ${url} (${res.status}).`;
            return [];
        }
        const text = await res.text();
        return d3.csvParse(text, (row) => ({
            tconst: row.tconst ?? "",
            title_type: row.title_type ?? "",
            primary_title: row.primary_title ?? "",
            original_title: row.original_title ?? "",
            year: row.year ? new Date(parseInt(row.year, 10), 0, 1) : new Date(0),
            runtime_minutes: row.runtime_minutes ? parseInt(row.runtime_minutes, 10) || 0 : 0,
            genres: row.genres ? row.genres.split(",").map((g) => g.trim()) : [],
            average_rating: row.average_rating ? parseFloat(row.average_rating) || 0 : 0,
            num_votes: row.num_votes ? parseInt(row.num_votes, 10) || 0 : 0,
        })) as TMovie[];
    }

    const YEAR_X_RANGE: [number, number] = [-1050, 1050];
    const STACK_Z = -40;
    const MAX_CHIPS = 28;
    const CHIP_WORLD = 8.2;

    function estimateStackBounds(stacks: YearStack[]): { top: number; halfX: number } {
        const halfX = Math.abs(YEAR_X_RANGE[0]) + 120;
        if (stacks.length === 0) return { top: FLOOR + 220, halfX };

        const maxPerLayer = Math.max(
            1,
            d3.max(stacks, (d) => Math.max(d.comedy, d.romance, d.drama)) ?? 1,
        );

        let peak = FLOOR;
        for (const row of stacks) {
            let y = FLOOR;
            let rowTop = FLOOR;
            for (const layer of LAYERS) {
                const c = row[layer.key];
                if (c <= 0) continue;
                if (c <= MAX_CHIPS) {
                    for (let i = 0; i < c; i++) {
                        y += CHIP_WORLD * 1.06;
                        const cy = y - CHIP_WORLD * 0.35;
                        rowTop = Math.max(rowTop, cy + CHIP_WORLD);
                    }
                    y += 6;
                } else {
                    const h = 16 + (c / maxPerLayer) * 400;
                    rowTop = Math.max(rowTop, y + h);
                    y += h + 6;
                }
            }
            peak = Math.max(peak, rowTop);
        }
        return { top: peak, halfX };
    }

    function fitCameraToStacks(
        cam: THREE.PerspectiveCamera,
        screenW: number,
        screenH: number,
        stacks: YearStack[],
    ) {
        const { top: dataTop, halfX } = estimateStackBounds(stacks);
        const titleTop = FLOOR + 520 + 55;
        const sceneTop = Math.max(dataTop, titleTop) + 90;
        const sceneBottom = FLOOR - 60;
        const spanY = sceneTop - sceneBottom;
        const midY = (sceneBottom + sceneTop) / 2;
        const zFocus = STACK_Z;

        const fovDeg = 33;
        const vfov = THREE.MathUtils.degToRad(fovDeg);
        const aspect = screenW / screenH;
        const hfov = 2 * Math.atan(Math.tan(vfov / 2) * aspect);
        const margin = 1.22;
        const distV = (spanY / 2 / Math.tan(vfov / 2)) * margin;
        const distH = (halfX / Math.tan(hfov / 2)) * margin;
        const dist = Math.max(distV, distH, 2100);

        cam.fov = fovDeg;
        cam.aspect = aspect;
        cam.near = 8;
        cam.far = 25000;
        cam.up.set(0, 1, 0);
        cam.position.set(0, midY, zFocus + dist);
        cam.lookAt(0, midY, zFocus);
        cam.updateProjectionMatrix();
    }

    onMount(() => {
        void (async () => {
            const movies = await loadMovies();
            const yearStacks = buildYearStacks(movies);
            init(window.innerWidth, window.innerHeight, yearStacks);
        })();
    });

    function init(SCREEN_WIDTH: number, SCREEN_HEIGHT: number, yearStacks: YearStack[]) {
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(SCREEN_WIDTH, SCREEN_HEIGHT);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFShadowMap;
        container.appendChild(renderer.domElement);

        camera = new THREE.PerspectiveCamera(33, SCREEN_WIDTH / SCREEN_HEIGHT, 8, 25000);
        framingStacks = yearStacks;
        fitCameraToStacks(camera, SCREEN_WIDTH, SCREEN_HEIGHT, yearStacks);

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
        light.position.set(520, 2100, 1350);
        light.castShadow = true;
        Object.assign(light.shadow.camera, {
            top: 2400,
            bottom: -2400,
            left: -2400,
            right: 2400,
            near: 500,
            far: 4500,
        });
        light.shadow.bias = 0.0001;
        light.shadow.mapSize.width = 2048;
        light.shadow.mapSize.height = 1024;
        scene.add(light);

        addGround(scene, FLOOR, `${base}/3d/grasslight-big.jpg`);

        const fontLoader = new FontLoader();
        fontLoader.load(
            `${base}/3d/helvetiker_bold.typeface.json`,
            (font: Font) => {
                const textGeo = new TextGeometry("summer movies", {
                    font,
                    size: 40,
                    depth: 15,
                });
                textGeo.computeBoundingBox();
                const bb = textGeo.boundingBox!;
                const cx = -0.5 * (bb.max.x - bb.min.x);
                const titleMesh = new THREE.Mesh(
                    textGeo,
                    new THREE.MeshStandardMaterial({ color: 0x449900 }),
                );
                titleMesh.position.set(cx, FLOOR + 520, -120);
                titleMesh.castShadow = true;
                scene.add(titleMesh);

                createYearStacks(scene, font, yearStacks);
            },
            undefined,
            () => {
                loadError = "Font failed to load (check static/3d/helvetiker_bold.typeface.json).";
            },
        );

        const models = [
            {
                path: `${base}/3d/Horse.glb`,
                speed: 300,
                duration: 1,
                x: 100 - Math.random() * 1000,
                y: FLOOR,
                z: 300,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Horse.glb`,
                speed: 300,
                duration: 1,
                x: 100 - Math.random() * 1000,
                y: FLOOR,
                z: 450,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Flamingo.glb`,
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 100,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Flamingo.glb`,
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 200,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Parrot.glb`,
                speed: 350,
                duration: 0.5,
                x: 500 - Math.random() * 500,
                y: FLOOR + 500,
                z: 700,
                scale: 0.5,
            },
        ];
        mixer = loadModels(models, scene, mixer, morphs);

        window.addEventListener("resize", () => {
            onWindowResize(camera, renderer, window.innerWidth, window.innerHeight);
            fitCameraToStacks(camera, window.innerWidth, window.innerHeight, framingStacks);
        });

        renderer.setAnimationLoop(animate);
    }

    function createYearStacks(scene: THREE.Scene, font: Font, yearStacks: YearStack[]) {
        if (yearStacks.length === 0) return;

        const years = yearStacks.map((d) => d.year);
        const xScale = d3
            .scaleBand<number>()
            .domain(years)
            .range(YEAR_X_RANGE)
            .padding(0.12);

        const maxPerLayer = Math.max(
            1,
            d3.max(yearStacks, (d) => Math.max(d.comedy, d.romance, d.drama)) ?? 1,
        );

        const bw = xScale.bandwidth();
        const depth = Math.min(70, bw * 0.95);

        for (const row of yearStacks) {
            const x0 = xScale(row.year);
            if (x0 === undefined) continue;
            const cx = x0 + bw / 2;
            const z0 = STACK_Z;

            let y = FLOOR;
            for (const layer of LAYERS) {
                const count = row[layer.key];
                const mat = new THREE.MeshStandardMaterial({
                    color: layer.color,
                    roughness: 0.45,
                    metalness: 0.08,
                });
                if (count <= 0) continue;

                if (count <= MAX_CHIPS) {
                    const geo = CHIP_GEOM[layer.key];
                    const n = count;
                    for (let i = 0; i < n; i++) {
                        const mesh = new THREE.Mesh(geo, mat);
                        y += CHIP_WORLD * 1.06;
                        mesh.scale.setScalar(CHIP_WORLD);
                        mesh.position.set(cx, y - CHIP_WORLD * 0.35, z0);
                        mesh.castShadow = true;
                        mesh.receiveShadow = true;
                        scene.add(mesh);
                    }
                    y += 6;
                } else {
                    const h = 16 + (count / maxPerLayer) * 400;
                    const mesh = new THREE.Mesh(
                        new THREE.BoxGeometry(bw * 0.62, h, depth),
                        mat,
                    );
                    mesh.position.set(cx, y + h / 2, z0);
                    mesh.castShadow = true;
                    mesh.receiveShadow = true;
                    scene.add(mesh);
                    y += h + 6;
                }
            }

            const showYear =
                years.length <= 18 ||
                row.year % 5 === 0 ||
                row.year === years[0] ||
                row.year === years[years.length - 1];
            if (showYear) {
                const label = new TextGeometry(String(row.year), {
                    font,
                    size: 11,
                    depth: 2,
                });
                label.computeBoundingBox();
                const lb = label.boundingBox!;
                const lx = -0.5 * (lb.max.x - lb.min.x);
                const lz = -0.5 * (lb.max.z - lb.min.z);
                const tm = new THREE.Mesh(
                    label,
                    new THREE.MeshPhysicalMaterial({ color: 0xf8f8f8 }),
                );
                tm.position.set(cx + lx, FLOOR + 12, z0 + depth * 0.35 + lz);
                tm.castShadow = true;
                scene.add(tm);
            }
        }
    }

    function animate() {
        const delta = clock.getDelta();
        mixer.update(delta);
        morphs.forEach((morph) => {
            const m = morph as THREE.Mesh & { speed: number };
            m.position.x += m.speed * delta;
            if (m.position.x > window.innerWidth / 2) {
                m.position.x = -window.innerWidth / 2 - Math.random() * 200;
            }
        });
        renderer.render(scene, camera);
    }
</script>

<div class="page">
    {#if loadError}
        <p class="banner">{loadError}</p>
    {/if}
    <aside class="legend">
        <h4>Legend</h4>
        <ul>
            {#each LAYERS as L}
                <li>
                    <span class="swatch" style:background={L.swatch}></span>
                    {L.label}
                </li>
            {/each}
        </ul>
    </aside>
    <div bind:this={container} class="container"></div>
</div>

<style>
    .page {
        position: relative;
        width: 100vw;
        height: 100vh;
    }
    .container {
        width: 100%;
        height: 100%;
    }
    .banner {
        position: absolute;
        top: 0.5rem;
        left: 50%;
        transform: translateX(-50%);
        z-index: 2;
        margin: 0;
        padding: 0.35rem 0.75rem;
        background: #2c1810;
        color: #f5e6d3;
        font-size: 0.85rem;
    }
    .legend {
        position: absolute;
        top: 0.75rem;
        right: 0.75rem;
        z-index: 2;
        padding: 0.5rem 0.75rem 0.65rem;
        background: rgba(255, 255, 255, 0.88);
        border-radius: 4px;
        font-size: 0.8rem;
        box-shadow: 0 1px 6px rgba(0, 0, 0, 0.12);
    }
    .legend h4 {
        margin: 0 0 0.35rem;
        font-size: 0.85rem;
        font-weight: 600;
    }
    .legend ul {
        list-style: none;
        margin: 0;
        padding: 0;
    }
    .legend li {
        display: flex;
        align-items: center;
        gap: 0.4rem;
        margin-bottom: 0.2rem;
    }
    .swatch {
        width: 14px;
        height: 14px;
        border-radius: 3px;
        flex-shrink: 0;
    }
    :global(body) {
        margin: 0;
        overflow: hidden;
    }
</style>

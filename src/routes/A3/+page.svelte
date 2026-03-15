<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { FontLoader, Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";

    import { addGround, onWindowResize, loadModels } from "$lib/Helper-3D";

    import * as d3 from "d3";
    import { base } from "$app/paths";
    import type { TMovie } from "../../types";

    type GenreData = { [key: string]: number };

    let container: HTMLElement;
    let camera: THREE.PerspectiveCamera;
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    const FLOOR = -250;
    const morphs: THREE.Mesh[] = [];
    let mixer: THREE.AnimationMixer;
    const clock = new THREE.Clock();

    function getGenreCounts(movies: TMovie[]): GenreData {
        const res: GenreData = {};
        movies.forEach((movie) => {
            movie.genres.filter((g) => g && g !== "NA").forEach((genre: string) => {
                res[genre] = (res[genre] || 0) + 1;
            });
        });
        return res;
    }

    async function loadMovies(): Promise<TMovie[]> {
        const csvUrl = `${base}/summer_movies.csv`;
        return await d3.csv(csvUrl, (row) => ({
            tconst: row.tconst ?? "",
            title_type: row.title_type ?? "",
            primary_title: row.primary_title ?? "",
            original_title: row.original_title ?? "",
            year: row.year ? new Date(parseInt(row.year), 0, 1) : new Date(0),
            runtime_minutes: row.runtime_minutes ? parseInt(row.runtime_minutes) || 0 : 0,
            genres: row.genres ? row.genres.split(",").map((g) => g.trim()) : [],
            average_rating: row.average_rating ? parseFloat(row.average_rating) || 0 : 0,
            num_votes: row.num_votes ? parseInt(row.num_votes) || 0 : 0,
        } as TMovie));
    }

    onMount(async () => {
        const movies = await loadMovies();
        const genreData = getGenreCounts(movies);
        const sortedGenres = Object.keys(genreData).sort((a, b) => genreData[b] - genreData[a]);
        const topGenres = sortedGenres.slice(0, 3);
        const data: GenreData = {};
        topGenres.forEach((g) => { data[g] = genreData[g]; });
        init(window.innerWidth, window.innerHeight, data);
    });

    function init(SCREEN_WIDTH: number, SCREEN_HEIGHT: number, genreData: GenreData) {
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(SCREEN_WIDTH, SCREEN_HEIGHT);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFShadowMap;
        container.appendChild(renderer.domElement);

        camera = new THREE.PerspectiveCamera(
            23,
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            3000,
        );
        camera.position.set(0, 80, 1800);
        camera.lookAt(0, -60, 0);

        scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87ceeb);
        new THREE.TextureLoader().load(`${base}/3d/sky.jpg`, (texture) => {
            texture.repeat.set(0.8, 1);
            scene.background = texture;
        });

        const ambient = new THREE.AmbientLight(0xffffff);
        scene.add(ambient);

        const light = new THREE.DirectionalLight(0xffffff, 3);
        light.position.set(0, 1500, 1000);
        light.castShadow = true;
        Object.assign(light.shadow.camera, {
            top: 2000,
            bottom: -2000,
            left: -2000,
            right: 2000,
            near: 1200,
            far: 2500,
        });
        light.shadow.bias = 0.0001;
        light.shadow.mapSize.width = 2048;
        light.shadow.mapSize.height = 1024;
        scene.add(light);

        addGround(scene, FLOOR, `${base}/3d/grasslight-big.jpg`);

        const fontLoader = new FontLoader();
        fontLoader.load(`${base}/3d/helvetiker_bold.typeface.json`, (font: Font) => {
            const textGeo = new TextGeometry("summer movies", {
                font: font,
                size: 40,
                depth: 15,
            });
            textGeo.computeBoundingBox();
            const centerOffset = -0.5 * (textGeo!.boundingBox!.max.x - textGeo!.boundingBox!.min.x);
            const textMaterial = new THREE.MeshStandardMaterial({ color: 0x449900 });
            const titleMesh = new THREE.Mesh(textGeo, textMaterial);
            titleMesh.position.x = centerOffset;
            titleMesh.position.y = FLOOR + 550;
            titleMesh.position.z = -350;
            titleMesh.castShadow = true;
            scene.add(titleMesh);

            createBars(scene, font, genreData);
        });

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
                y: FLOOR + 350,
                z: 100,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Flamingo.glb`,
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 350,
                z: 200,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Parrot.glb`,
                speed: 350,
                duration: 0.5,
                x: 500 - Math.random() * 500,
                y: FLOOR + 350,
                z: 700,
                scale: 0.5,
            },
        ];
        mixer = loadModels(models, scene, mixer, morphs);

        window.addEventListener("resize", () =>
            onWindowResize(camera, renderer, window.innerWidth, window.innerHeight),
        );

        renderer.setAnimationLoop(animate);
    }

    function createBars(scene: THREE.Scene, font: Font, genreData: GenreData) {
        const maxHeight = 400;
        const barMaxWidth = 550;

        const xScale = d3
            .scaleBand()
            .domain(Object.keys(genreData))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.1);

        const maxVal = Math.max(...Object.values(genreData), 1);
        const yScale = d3.scaleLinear().domain([0, maxVal]).range([0, maxHeight]);

        Object.keys(genreData).forEach((genre) => {
            const bar = createOneBar(yScale(genreData[genre]), xScale.bandwidth());
            bar.position.set(
                (xScale(genre) ?? 0) + xScale.bandwidth() / 2,
                FLOOR + yScale(genreData[genre]) / 2,
                0,
            );
            scene.add(bar);

            addLabelToBar(
                scene,
                `${genre}: ${genreData[genre]}`,
                (xScale(genre) ?? 0) - xScale.bandwidth() / 2,
                FLOOR + yScale(genreData[genre]) + 45,
                xScale.bandwidth(),
                font,
            );
        });
    }

    function createOneBar(height: number, width: number) {
        const geometry = new THREE.CylinderGeometry(width / 2, width / 2, height, 32);
        const material = new THREE.MeshStandardMaterial({
            map: new THREE.TextureLoader().load(`${base}/3d/wood-texture.jpg`),
        });
        const bar = new THREE.Mesh(geometry, material);
        bar.castShadow = true;
        bar.receiveShadow = true;
        return bar;
    }

    function addLabelToBar(
        scene: THREE.Scene,
        text: string,
        x: number,
        y: number,
        z: number,
        font: Font,
    ) {
        const textGeometry = new TextGeometry(text, {
            font: font,
            size: 12,
            depth: 4,
        });
        const textMaterial = new THREE.MeshPhysicalMaterial({ color: 0xffffff });
        const textMesh = new THREE.Mesh(textGeometry, textMaterial);
        textMesh.position.set(x, y, z);
        textMesh.castShadow = true;
        textMesh.receiveShadow = false;
        scene.add(textMesh);
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

<div bind:this={container} class="container"></div>

<style>
    :global(body) {
        margin: 0;
        overflow: hidden;
    }
    div.container {
        width: 100vw;
        height: 100vh;
    }
</style>

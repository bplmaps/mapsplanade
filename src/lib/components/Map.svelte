<script>
  import { onMount } from "svelte";
  import OlMap from "ol/Map";
  import View from "ol/View";
  import GeoJSON from "ol/format/GeoJSON";
  import VectorSource from "ol/source/Vector";
  import VectorLayer from "ol/layer/Vector";
  import Style from "ol/style/Style";
  import Stroke from "ol/style/Stroke";
  import Fill from "ol/style/Fill";
  import { useGeographic } from "ol/proj";
  import Tile from "ol/layer/Tile";
  import { TileJSON } from "ol/source";
  import Select from "ol/interaction/Select";
  import { click, pointerMove } from "ol/events/condition";
  import BibliographicInfo from "./BibliographicInfo.svelte";

  let title = $state();
  let creator = $state();
  let date = $state(); // width, height;
  let hovered = $state(false);

  useGeographic();

  let map;
  const mtkey = "xzHYzv10Mfc1eJ8Vbizl";

  // Dynamic style function using `id`
  const patternCache = new Map();

  // onMount(async () => {
  //   // if (!hoveredId) return;
  //   const res = await fetch(
  //     `https://collections.leventhalmap.org/search/${id}.json`
  //   );

  //   const item = await res.json();
  //   console.log(item);
  //   // title = item.title_primary_;
  //   // creator = item.creator;
  //   // date = item.date;
  // });

  function loadThumbnailPattern(id) {
    return new Promise((resolve) => {
      if (patternCache.has(id)) return resolve(patternCache.get(id));

      const img = new Image();
      img.crossOrigin = "anonymous";
      img.src = `https://s3.us-east-2.wasabisys.com/lmec-public-files/thumbnails/${id}.png`;

      img.onload = () => {
        const canvas = document.createElement("canvas");
        const ctx = canvas.getContext("2d");
        canvas.width = img.height;
        canvas.height = img.width;
        ctx.drawImage(img, 0, 0);
        const pattern = ctx.createPattern(img, "repeat");
        ctx.fillRect(0, 0, 300, 300);
        if (pattern) {
          patternCache.set(id, pattern);
          resolve(pattern);
        } else {
          resolve(null);
        }
      };

      img.onerror = () => {
        // console.warn(`Failed to load thumbnail for ${id}`);
        resolve(null);
      };
    });
  }

  function hoverStyleFunction(feature) {
    const id = feature.get("id");
    const pattern = patternCache.get(id);

    return new Style({
      fill: new Fill({ color: pattern || "#cccccc" }),
      stroke: new Stroke({ color: "red", width: 2 }),
    });
  }

  const baseStyle = new Style({
    fill: new Fill({ color: "#eeeeee" }),
  });

  onMount(async () => {
    async function getData(hoveredId) {
      const res = await fetch(
        `https://collections.leventhalmap.org/search/${hoveredId}.json`
      );
      const item = await res.json();
      console.log(item)
      title = item["response"]["document"]["title_info_primary_tsi"];
      creator = item["response"]["document"]["name_tsim"];
      date = item["response"]["document"]["date_tsim"][0];
    }

    const bboxes = await fetch("/assets/bboxes.geojson").then((r) => r.json());

    let features = new GeoJSON().readFeatures(bboxes, {
      featureProjection: "EPSG:4326",
    });

    // only use the first 100 features of while testing

    features = features.slice(0, 100);

    const featureIds = features.map((f) => f.get("id"));

    await loadThumbnailPattern(featureIds);

    const vectorSource = new VectorSource({ features });

    map = new OlMap({
      target: "map",
      controls: [],
      layers: [
        new Tile({
          source: new TileJSON({
            url: `https://api.maptiler.com/maps/satellite/tiles.json?key=${mtkey}`,
            tileSize: 512,
            crossOrigin: "anonymous",
          }),
        }),
        new VectorLayer({
          source: vectorSource,
          style: (feature) =>
            new Style({
              fill: new Fill({ color: feature.get("COLOR") || "#eeeeee" }),
            }),
        }),
      ],
      view: new View({
        center: [-71.062608967, 42.352372528],
        zoom: 30,
        projection: "EPSG:4326",
      }),
    });

    // Hover interaction

    const hoverSelect = new Select({
      condition: pointerMove,
      style: (feature) => {
        const id = feature.get("id");
        getData(id);

        if (!patternCache.has(id)) {
          loadThumbnailPattern(id).then(() => {
            map.render();
          });
        }

        return new Style({
          fill: new Fill({ color: patternCache.get(id) || "#eeeeee" }),
          stroke: new Stroke({ color: "red", width: 2 }),
          zIndex: 1001,
        });
      },
    });

    map.on("moveend", () => {
      const extent = map.getView().calculateExtent(map.getSize());
      vectorSource.forEachFeatureInExtent(extent, (feature) => {
        const id = feature.get("id");
        if (id && !patternCache.has(id)) {
          loadThumbnailPattern(id); // preload silently
        }
      });
    });

    // Click interaction
    const clickSelect = new Select({
      condition: click,
      style: hoverStyleFunction,
    });

    try {
      map.addInteraction(hoverSelect);
      map.addInteraction(clickSelect);
    } catch (error) {
      console.log(error);
    }

    clickSelect.on("select", (e) => {
      const selected = e.selected;
      if (selected.length) {
        const id = selected[0].get("id");
        window.open(
          `https://collections.leventhalmap.org/search/${id}`,
          "_blank"
        );
      }
    });
  });
</script>

<div id="map" class="50-vw"></div>
<div class="m-4 p-4 bg-gray-100">
  <div class="flex">
    <div class="w-1/6 text-lg font-semibold">Title:</div>
    <div>{title}</div>
  </div>
  <div class="flex">
    <div class="w-1/6 text-lg font-semibold">Creator:</div>
    <div>{creator}</div>
  </div>
  <div class="flex">
    <div class="w-1/6 text-lg font-semibold">Date:</div>
    <div>{date}</div>
  </div>

  <!-- <div class="font-medium">Width:</div>
  <div>{width}</div>

  <div class="font-medium">Height:</div>
  <div>{height}</div> -->
</div>

<style>
  #map {
    height: 50vh;
  }
</style>

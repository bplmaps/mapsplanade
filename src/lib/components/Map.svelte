<script>
  import { hovered } from "$lib/stores.js";
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
  import { TileJSON } from "ol/source";
  import Select from "ol/interaction/Select";
  import { click, pointerMove } from "ol/events/condition";
  import BibliographicInfo from "./BibliographicInfo.svelte";
  import TileLayer from "ol/layer/Tile";

  let item = $state();

  useGeographic();

  let map;
  const mtkey = "xzHYzv10Mfc1eJ8Vbizl";

  const canvasCache = new Map();

  async function loadThumbnailPattern(id, width = 512, height = 512) {
    if (canvasCache.has(id)) return canvasCache.get(id);

    return new Promise((resolve) => {
      const img = new Image();
      img.crossOrigin = "anonymous";
      img.src = `https://s3.us-east-2.wasabisys.com/lmec-public-files/thumbnails/${id}.png`;

      img.onload = () => {
        const canvas = document.createElement("canvas");
        canvas.width = width;
        canvas.height = height;

        const ctx = canvas.getContext("2d");
        ctx.drawImage(img, 0, 0, width, height);
        canvasCache.set(id, canvas);
        resolve(canvas);
      };

      img.onerror = () => {
        console.warn(`Thumbnail failed to load: ${id}`);
        resolve(null);
      };
    });
  }

  function baseStyleFunction(feature) {
    const id = feature.get("id");
    const fillPattern = canvasCache.get(id) || "#eeeeee";
    return new Style({
      fill: new Fill({ color: fillPattern }),
    });
  }

  function hoverStyleFunction(feature) {
    const id = feature.get("id");
    const canvas = canvasCache.get(id);
    if (!canvas) return null;

    const base = thumbnailStyleFunction(feature);

    return new Style({
      renderer: base.getRenderer(),
      stroke: new Stroke({ color: "white", width: 4 }),
      zIndex: 10,
    });
  }

  function clickStyleFunction(feature) {
    const id = feature.get("id");
    const pattern = canvasCache.get(id);
    return new Style({
      fill: new Fill({ color: pattern || "#cccccc" }),
      stroke: new Stroke({ color: "white", width: 4 }),
    });
  }

  function thumbnailStyleFunction(feature) {
    const id = feature.get("id");
    const canvas = canvasCache.get(id);
    if (!canvas) return null;

    return new Style({
      renderer: (pixelCoords, renderState) => {
        const ctx = renderState.context;
        if (!pixelCoords || pixelCoords.length === 0) return;

        ctx.save();
        ctx.beginPath();

        pixelCoords.forEach((ring) => {
          ring.forEach(([x, y], i) => {
            if (i === 0) ctx.moveTo(x, y);
            else ctx.lineTo(x, y);
          });
        });

        ctx.clip();

        // Compute bounding box in pixel space
        let minX = Infinity,
          minY = Infinity,
          maxX = -Infinity,
          maxY = -Infinity;

        pixelCoords[0].forEach(([x, y]) => {
          if (x < minX) minX = x;
          if (y < minY) minY = y;
          if (x > maxX) maxX = x;
          if (y > maxY) maxY = y;
        });

        const width = maxX - minX;
        const height = maxY - minY;

        // Draw the cached thumbnail image to fit polygon bbox
        ctx.drawImage(canvas, minX, minY, width, height);
        ctx.restore();
      },
    });
  }

  onMount(async () => {
    async function getData(id) {
      const res = await fetch(
        `https://collections.leventhalmap.org/search/${id}.json`
      );
      item = await res.json();
    }

    const bboxes = await fetch("/assets/bboxes_20250729.geojson").then((r) =>
      r.json()
    );

    let features = new GeoJSON().readFeatures(bboxes, {
      featureProjection: "EPSG:4326",
    });

    // only use the first 100 features of while testing

    features = features.slice(0, 500);

    const featureIds = features.map((f) => f.get("id"));

    await Promise.all(
      features.map(async (feature) => {
        const id = feature.get("id");
        await loadThumbnailPattern(id);
        feature.setStyle(thumbnailStyleFunction(feature));
      })
    );

    const vectorSource = new VectorSource({ features });

    map = new OlMap({
      controls: [],
      layers: [
        new TileLayer({
          source: new TileJSON({
            url: `https://api.maptiler.com/maps/satellite/tiles.json?key=${mtkey}`,
            tileSize: 512,
            crossOrigin: "anonymous",
          }),
        }),
        new VectorLayer({
          source: vectorSource,
          style: baseStyleFunction,
        }),
      ],
      target: "map",
      view: new View({
        center: [-71.062608967, 42.352372528],
        zoom: 30,
        projection: "EPSG:4326",
      }),
    });

    let previousHovered = null;
    map.on("pointermove", function (e) {
      var pixel = map.getEventPixel(e.originalEvent);
      var hit = map.hasFeatureAtPixel(pixel);
      map.getViewport().style.cursor = hit ? "pointer" : "";
    });
    map.on("pointermove", (e) => {
      let found = false;

      map.forEachFeatureAtPixel(e.pixel, (feature) => {
        hovered.set(feature);
        found = true;

        const id = feature.get("id");
        getData(id);

        if (previousHovered && previousHovered !== feature) {
          previousHovered.setStyle(thumbnailStyleFunction(previousHovered));
        }

        feature.setStyle(hoverStyleFunction(feature)); // this isn't working
        previousHovered = feature;

        return true;
      });

      if (!found) {
        if (previousHovered) {
          // ❗ Restore style here too
          previousHovered.setStyle(thumbnailStyleFunction(previousHovered));
          previousHovered = null;
        }
        hovered.set(null);
        item = null;
      }
    });

    map.on("moveend", () => {
      const extent = map.getView().calculateExtent(map.getSize());
      vectorSource.forEachFeatureInExtent(extent, (feature) => {
        const id = feature.get("id");
        if (id && !canvasCache.has(id)) {
          loadThumbnailPattern(id);
        }
      });
    });

    // Initial preload
    features.forEach((feature) => {
      const id = feature.get("id");
      loadThumbnailPattern(id);
    });

    const clickSelect = new Select({
      condition: click,
      style: clickStyleFunction,
    });

    try {
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

<BibliographicInfo />

<div class="w-full">
  <div id="map"></div>
</div>

<style>
  #map {
    width: 100%;
    height: 60vh;
  }
</style>

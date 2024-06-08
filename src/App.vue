<script setup lang="ts">
import { onMounted, ref } from 'vue';
import {
  Ion,
  Viewer,
  Cesium3DTileset,
  Color,
  Terrain,
  ScreenSpaceEventType,
  ScreenSpaceEventHandler,
  defined,
  Label,
  LabelCollection,
  Cartesian3,
  VerticalOrigin,
  Cesium3DTileStyle,
  HorizontalOrigin,
  HeadingPitchRoll,
  Matrix4,
  PostProcessStageLibrary
} from 'cesium';
import "cesium/Build/Cesium/Widgets/widgets.css";

window.CESIUM_BASE_URL = '/cesium/';

const viewer =  ref<Viewer>();
const tileset = ref<Cesium3DTileset>();
const features = ref<any>();
const annotations = ref<LabelCollection>();
const isRandomizeColor = ref<boolean>(false);
const isShowAnnotations = ref<boolean>(false);

// Set the initial camera view to look at Manhattan
const initialPosition = Cartesian3.fromDegrees(
  -74.01881302800248,
  40.69114333714821,
  753
);
const initialOrientation = HeadingPitchRoll.fromDegrees(
  21.27879878293835,
  -21.34390550872461,
  0.0716951918898415
);

const defaultTilesColor = "rgba(255, 255, 255, 1)";

const silhouetteBlue = PostProcessStageLibrary.createEdgeDetectionStage();
silhouetteBlue.uniforms.color = Color.BLUE;
silhouetteBlue.uniforms.length = 0.01;
silhouetteBlue.selected = [];

onMounted(async () => {
  Ion.defaultAccessToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJjYjIyMDQxYi1hZmZlLTQ3NWQtOWQxOS1mNTJkYTJmZGJkNmIiLCJpZCI6MjEzODIyLCJpYXQiOjE3MTUxODA2MDl9.96azLFzedet5YIC4irMkExjnqVKM9kRT_uCyg_HwKfU';

  viewer.value = new Viewer('cesiumWrap', {
    terrain: Terrain.fromWorldTerrain(),
  });

  viewer.value.scene.camera.setView({
    destination: initialPosition,
    orientation: initialOrientation,
    endTransform: Matrix4.IDENTITY,
  });

  viewer.value.scene.postProcessStages.add(
    PostProcessStageLibrary.createSilhouetteStage([
      silhouetteBlue
    ])
  );

  try {
    tileset.value = await Cesium3DTileset.fromIonAssetId(75343, { skipLevelOfDetail: true });
    viewer.value.scene.primitives.add(tileset.value);
    tileset.value.style = new Cesium3DTileStyle({
      color: defaultTilesColor,
    });

    features.value = getTileFeatures(tileset.value);
  } catch (error) {
    console.error(`Error loading tileset: ${error}`);
  }

  // Information about the currently highlighted feature
  const highlighted: {feature: any, originalColor: Color} = {
    feature: undefined,
    originalColor: new Color(),
  };

  // Color a feature yellow on hover.
  viewer.value.screenSpaceEventHandler.setInputAction((movement: ScreenSpaceEventHandler.MotionEvent) => {
    if (isRandomizeColor.value) {
      // If a feature was previously highlighted, undo the highlight
      if (defined(highlighted.feature)) {
        highlighted.feature.color = highlighted.originalColor;
        highlighted.feature = undefined;
      }

      // Pick a new feature
      const pickedFeature = viewer.value?.scene.pick(movement.endPosition);
      if (!defined(pickedFeature) || pickedFeature.primitive instanceof Label) {
        return;
      }

      // Highlight the feature
      highlighted.feature = pickedFeature;
      Color.clone(pickedFeature.color, highlighted.originalColor);
      pickedFeature.color = Color.YELLOW;
    }
  }, ScreenSpaceEventType.MOUSE_MOVE);

  annotations.value = viewer.value.scene.primitives.add(new LabelCollection());
})

const toggleRandomize = (event: Event) => {
  isRandomizeColor.value = (event.target as HTMLInputElement).checked;
  silhouetteBlue.selected = [];

  features.value.forEach((feature: any) => {
    feature.color = isRandomizeColor.value
      ? Color.fromRandom({ alpha: 1.0 })
      : Color.WHITE;
    // TODO for border color
    //isRandomizeColor.value && silhouetteBlue.selected.push(feature)
  });

}

const toggleAnnotations = (event: Event) => {
  isShowAnnotations.value = (event.target as HTMLInputElement).checked;

  if (isShowAnnotations.value) {
    features.value.forEach((feature: any) => {
      const position: Cartesian3 = feature.content._model.boundingSphere.center;

      if (position) {
        annotations.value?.add({
          position,
          text: new Date().toDateString(),
          showBackground: true,
          font: "10px monospace",
          horizontalOrigin: HorizontalOrigin.LEFT,
          verticalOrigin: VerticalOrigin.BOTTOM,
          disableDepthTestDistance: Number.POSITIVE_INFINITY,
        });
      }
    });
  } else {
    annotations.value?.removeAll();
  }
}

const getTileFeatures = (tileset: Cesium3DTileset) => {
  const features: any = [];

  tileset.tileVisible.addEventListener(function (tile) {
    const content = tile.content;
    const featuresLength = content.featuresLength;

    for (let i = 0; i < featuresLength; i++) {
      features.push(content.getFeature(i));
    }
  });

  return features;
}

</script>

<template>
  <div><label><input type="checkbox" name="" id="" @change="toggleRandomize">Randomize It!</label></div>
  <div><label><input type="checkbox" name="" id="" @change="toggleAnnotations">Show Annotations</label></div>
  <div id="cesiumWrap"></div>
</template>

<style scoped>
#cesiumWrap {
  width: 100%; 
  height: 85vh;
  margin-top: 10px;
}
</style>

# Thin Instance Buffer Management (WebGPU)

## Multiplayer WebGPU Buffer Fix

### Problem Description
When multiple players join the same zone, WebGPU validation errors occur:
```
WebGPU uncaptured error: Instance range (first: 0, count: 58) requires a larger buffer (928) than the bound buffer size (912) of the vertex buffer at slot 1 with stride 16.
```

This causes the first player's client to crash when a second player reaches character selection.

### Root Cause Analysis
The issue stems from the thin instance buffer management system in Babylon.js:

1. **Buffer Size Mismatch**: When `addThinInstance()` is called for new player entities, the GPU vertex buffer isn't properly resized to accommodate the new instance count.

2. **Concurrent Updates**: Multiple entities being created simultaneously can cause race conditions in buffer updates.

3. **Automatic Refresh Timing**: BabylonJS's automatic refresh mechanism conflicts with manual buffer updates.

### Technical Fix

#### 1. Fixed Thin Instance Buffer Management
**File**: `client/src/Game/Model/entity-cache.ts`

The implementation resizes the per-instance texture backing store whenever a new thin instance is added. This keeps the GPU-side buffer dimensions in sync with `thinInstanceCount` and avoids WebGPU validation errors.

```390:423:client/src/Game/Model/entity-cache.ts
const addThinInstance = (matrix: BJS.Matrix): number => {
  const instanceIdx = mergedMesh.thinInstanceAdd(matrix, true);
  const originalBuffer = (mergedMesh.metadata?.textureAttributeArray
    ?._texture?._bufferView ?? new Float32Array()) as Float32Array;
  const newWidth = submeshCount * mergedMesh.thinInstanceCount;
  const data = new Float32Array(4 * newWidth);
  data.set(originalBuffer, 0);
  mergedMesh.thinInstanceSetAttributeAt(
    'thinInstanceIndex',
    instanceIdx,
    [instanceIdx, 0],
    true,
  );
  if (mergedMesh.metadata.textureAttributeArray) {
    mergedMesh.metadata.textureAttributeArray.dispose();
  }
  const fresh = new BABYLON.RawTexture(
    data,
    newWidth,
    1,
    BABYLON.Constants.TEXTUREFORMAT_RGBA,
    scene,
    false,
    false,
    BABYLON.Constants.TEXTURE_NEAREST_NEAREST_MIPNEAREST,
    BABYLON.Constants.TEXTURETYPE_FLOAT,
  );
  mergedMesh.metadata.textureAttributeArray = fresh;
  return instanceIdx;
};
```

Where the per-frame matrix buffer flush occurs:

```548:567:client/src/Game/Model/entity-cache.ts
EntityCache.renderObserver = scene.onAfterRenderCameraObservable.add((camera) => {
  const meshes = new Set<BJS.Mesh>();
  for (const entity of EntityCache.entityInstances) {
    if (entity.hidden) {
      continue;
    }
    for (const mesh of entity.syncMatrix()) {
      meshes.add(mesh);
    }
  }
  for (const mesh of meshes) {
    mesh?.thinInstanceBufferUpdated('matrix');
  }
});
```

#### 2. Entity instantiation of thin instances
**File**: `client/src/Game/Model/entity.ts`

```361:367:client/src/Game/Model/entity.ts
const { mesh, addThinInstance } = this.entityContainer;
const thinInstanceIndex = addThinInstance(worldMat);
this.meshInstance = {
  mesh: mesh as BJS.Mesh,
  thinInstanceIndex,
};
```

#### 3. Notes on server-side spawns
This document focuses on client thin instance buffers. Server spawn logic is out of scope here.

### Movement smoothing
Not implemented in `client/src/Game/Zone/entity-pool.ts` at this time; the `process()` loop is currently empty and `UpdateSpawnPosition` applies positions/velocities directly.

### Additional notes

- Matrices are updated with `thinInstanceSetMatrixAt(..., /* noFlush= */ false)` in `Entity.syncMatrix`, then flushed once per frame in the render observer.
- The per-instance attribute `thinInstanceIndex` is set per instance add.

### Testing

Validated locally with multiple entities joining the scene using WebGPU; buffer sizes matched `thinInstanceCount` and no validation errors were observed during instance growth.

### Upstream Contribution Potential

This focuses on ensuring per-instance buffers are resized when `thinInstanceCount` grows, avoiding WebGPU validation errors in dynamic thin instance scenarios.

Consider contributing the thin instance buffer management improvements to the BabylonJS project as it resolves WebGPU validation errors in multi-entity scenarios.
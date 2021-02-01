# pcg_skel

Save time by generating robust neuronal skeletons directly from a ChunkedGraph dynamic segmentations!

## What you need:

For skeletons you just need a dynamic segmentation running on a [PyChunkedGraph](https://github.com/seung-lab/PyChunkedGraph) server.

To also use annotations, you also need a database backend running the [MaterializationEngine](https://github.com/seung-lab/MaterializationEngine).

## Key terms:

Ids in the PCG combine information about chunk level, spatial location, and unique object id.
This package uses the highest-resolution chunking, level 2, to derive neuronal topology and approximate spatial extent.
For clarity, I'll define a few terms:

* *L2 object*: Collection of attributes (supervoxels, mesh, etc) associated with a level 2 id.

* *L2 mesh*: The fragment of mesh associated with a level 2 id.

* *Chunk index/chunk index space*: The integer 3-d index of a chunk in the the grid that covers the volume, i.e. the chunk index space.

* *Euclidean position/Euclidean space*: A position in nanometers / biological space.

* *Chunk position*: The center of a specific chunk index in Euclidean space

* *Refinement*: Mapping the position of an L2 object from its chunk position to a more representitive point on the mesh.

## How to use:

There are a few potentially useful functions with progressively more features:

### chunk_index_skeleton
This function returns a meshparty skeleton with vertices given in chunk index space.

```python
def chunk_index_skeleton(root_id,
                         client=None,
                         datastack_name=None,
                         cv=None,
                         root_point=None,
                         invalidation_d=3,
                         return_mesh=False,
                         return_l2dict=False,
                         return_mesh_l2dict=False,
                         root_point_resolution=[4, 4, 40],
                         root_point_search_radius=300,
                         n_parallel=1):
```

_Notes:_

* A `FrameworkClient` or `datastack_name` must be provided, and a cloudvolume object is suggested for performance reasons, especially if running in bulk.

* If `root_point` is set, the function looks within `root_point_search_radius` for the closest point in the root_id segmentation and sets the associated level 2 ID as the root vertex. `root_point_search_radius` is in nanometers. `root_point` is in units of `root_point_resolution`, such that root_point * root_point_resolution should be correct in Euclidean space.

* `return_mesh` will also return the full graph of level 2 objects as a `meshparty.Mesh` that has no faces and uses only link edges. Vertices are in chunk index space.

* `return_l2dict` will also return a tuple of two dictionaries. The first, `l2dict`, is a dict mapping level 2 id to skeleton vertex index. This is one to many, since it includes passing through multiple level 2 ids that belong to the mesh and are collapsed into a common skeleton vertex. The second, `l2dict_reversed`, is a dict mapping skeleton vertex to level 2 id. This is one to one.

* `return_mesh_l2dict` will return an additional tuple of two dictionaries. The first maps level 2 ids to mesh vertices, the second mesh vertex index to level 2 id.

* `n_parallel` is passed directly to cloudvolume and can speed up mesh fragment downloading.

---

### refine_chunk_index_skeleton

Map a skeleton in chunk index space into Euclidean space.

```python
def refine_chunk_index_skeleton(
    sk_ch,
    l2dict_reversed,
    cv,
    refine_inds='all',
    scale_chunk_index=True,
    root_location=None,
    nan_rounds=20,
    return_missing_ids=False,
):
```

_Notes:_

* `sk_ch` is assumed to have positions in chunk index space.

* `l2dict_reversed` is the second dict that comes back in the tuple you get by setting `return_l2dict` to `True` in `chunk_index_skeleton`.

* `refine_inds` can be either `"all"` or a specific array of vertex indices. This parameter sets which vertices to refine. A handful of vertices can be processed very quickly, while a more accurate skeleton is generated by refining all vertices.

* `scale_chunk_index`, if True, will still map unrefined vertex indices from their chunk index into the chunk position in eucliden space.

* `nan_rounds` gives how many times to try to smoothly interpolate any vertices that did not get a position due to a missing L2 mesh fragment.

* `return_missing_ids` will also return a list of the ids of any missing L2 mesh fragment, for example if you want to re-run meshing on them.

---

### pcg_skeleton

Directly build a skeleton in Euclidean space, plus optionally handle soma points.

```python
def pcg_skeleton(root_id,
                 client=None,
                 datastack_name=None,
                 cv=None,
                 refine='all',
                 root_point=None,
                 root_point_resolution=[4, 4, 40],
                 root_point_search_radius=300,
                 collapse_soma=False,
                 collapse_radius=10_000.0,
                 invalidation_d=3,
                 return_mesh=False,
                 return_l2dict=False,
                 return_l2dict_mesh=False,
                 return_missing_ids=False,
                 nan_rounds=20,
                 n_parallel=1):
```

_Notes_:

* `refine` can take five values to determine which vertices to refine.
    1. `"all"`: Refine all vertices
    2. `"ep"`: Refine only end points.
    3. `"bp"`: Refine only branch points.
    4. `"bpep"` or `"epbp"`: Refine both branch and end points.
    5. `None`: Don't refine any vertex, but still map chunk positions coarsely.

    Options which download few mesh fragments are much faster than `"all"`.

* `collapse_soma`, if set to True, collapses all mesh vertices within `collapse_radius` into the root point.
Note that the root point is not a new point, but rather the closest level 2 object to the root point location.

---

### pcg_meshwork

Build a meshwork file with skeleton out from the level 2 graph.
Optionally, attach pre- and/or post-synaptic synapses.

```python
def pcg_meshwork(root_id,
                 datastack_name=None,
                 client=None,
                 cv=None,
                 refine='all',
                 root_point=None,
                 root_point_resolution=[4, 4, 40],
                 root_point_search_radius=300,
                 collapse_soma=False,
                 collapse_radius=DEFAULT_COLLAPSE_RADIUS,
                 synapses=None,
                 synapse_table=None,
                 remove_self_synapse=True,
                 invalidation_d=3,
                 n_parallel=4,
                 ):
```
_Notes_:

* The resulting meshwork file comes back with a "mesh" made out of the level 2 graph with vertices mapped to their chunk positions, a skeleton with `refine` and `collapse_soma` options as above, and one or more annotations.

* All resulting meshworks have the annotation `lvl2_ids`, which is based on a dataframe with column `lvl2_id` that has level 2 ids and `mesh_ind` that has the associated mesh index.
One can use the MeshIndex/SkeletonIndex properties like `nrn.anno.mesh_index.to_skel_index` to see the associated skeleton indices.

* If the `synapses` property is set to `"pre"`, `"post"`, or `"all"`, there is also an attempt to look up the root id's presynaptic synapses, postsynaptic synapses, or both (respectively).
Presynaptic synapses are in an annotation `"pre_syn"` and postsynaptic synapses are in an annotation called `"post_syn"`.

* If returning synapses, you must set the synapse table.
By default, synapses whose pre- and post-synaptic ids are both the same root id are excluded, but this can be turned off by setting `remove_self_synapse` to `False`.

## Example:

A minimal example to get the skeleton of an arbitrary neuron with root id `864691135761488438` and soma at the voxel-space location `253870, 236989, 20517` in the Minnie dataset:

```python
from annotationframeworkclient import FrameworkClient
import pcg_skel

client = FrameworkClient('minnie65_phase3_v1')

oid = 864691135407403977
root_point = [253870, 236989, 20517]
sk_l2 = pcg_skel.pcg_skeleton(oid,
                              client=client,
                              refine='all',
                              root_point=root_point,
                              root_point_resolution=[4,4,40],
                              collapse_soma=True,
                              n_parallel=8)
```

This took 78 seconds on my computer and generated a skeleton with 1310 vertices, all refined through the mesh fragments.
Most of the time is spent in refinement.
If you just select the `refine="epbp"` argument, it only refines the 79 branch and end points and accordingly takes a mere 12.5 seconds.
It is worth exploring sparse refinement options and interpolation if processing time is extremely important.

## To-dos: 

* Add cached l2 id lookups to never download the same fragments twice.

* Improve/document/test tooling for additional annotations.


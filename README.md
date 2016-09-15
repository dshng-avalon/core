### Pyblish Starter

Jumpstart your publishing pipeline with a basic configuration.

> WARNING: This is in development (see latest commit date) and is NOT YET ready for an audience

<br>
<br>

### Prerequisities

- Maya

<br>
<br>

### Limitations

- Doesn't illustrate cross-application features of Pyblish
- Sandboxed publishing

<br>
<br>

### Features

- Asset definition template
- Versioning

<br>
<br>

### Install

Starter takes the form of a Python package with embedded plug-ins.

```bash
$ pip install pyblish-starter
```

<br>
<br>

### Usage

Plug-ins are registered by calling `setup()`.

```python
>>> import pyblish_starter
>>> pyblish_starter.setup()
```

<br>
<br>

### Contract

Starter defines these families.

| Family              | Definition                                                  | Link
|:--------------------|:------------------------------------------------------------|:------------
| `starter.model`     | Geometry with deformable topology                           | [Spec](#startermodel)
| `starter.rig`       | An articulated `starter.model` for animators                | [Spec](#starterrig)
| `starter.animation` | Pointcached `starter.rig` for tech-anim and lighting        | [Spec](#starteranimation)

<br>

### `starter.model`

<img align="right" src="https://cloud.githubusercontent.com/assets/2152766/18526694/7453567a-7ab9-11e6-817c-84a874092399.png"></img>

A generic representation of geometry.

**Target Audience**

- Texturing
- Rigging
- Final render

**Requirements**

- Static geometry (no deformers, generators)
- One shape per transform `*`
- Zero transforms and pivots `*`
- No intermediate shapes `*`
- UVs within 0-1 with full coverage, no overlap `*`
- Unlocked normals `*`
- Manifold geometry `*`
- No edges with zero length `*`
- No faces with zero area `*`
- No self-intersections `*`

> `*` = Todo

**Data**

- `label (str, optional)`: Pretty printed name in graphical user interfaces

**Sets**

- `geometry_SEL (geometry)`: Meshes suitable for rigging
- `aux_SEL (any, optional)`: Auxilliary meshes for e.g. fast preview, collision geometry

<br>
<br>

### `starter.rig`

<img align="right" src="https://cloud.githubusercontent.com/assets/2152766/18526730/9c7f040a-7ab9-11e6-9007-4795ddbadde8.png"></img>

The `starter.rig` contains the necessary implementation and interface for animators to produce 

**Requirements**

- Channels in `controls_SEL` at *default* values`*`
- No input connection to animatable channel in `controls_SEL` `*`
- [No self-intersections on workout](#workout) `*`

**Data**

- `label (str, optional)`: Pretty printed name in graphical user interfaces

**Sets**

- `cache_SEL (geometry)`: Meshes suitable for pointcaching from animation
- `controls_SEL (transforms)`: All animatable controls
- `resources_SEL (any, optional)`: Nodes that reference an external file

<br>
<br>

### `starter.animation`

<img align="right" src="https://cloud.githubusercontent.com/assets/2152766/18526738/a0fba5ba-7ab9-11e6-934f-48ca2a2ce3d2.png"></img>

Point positions and normals represented as one Alembic file.

**Requirements**

- [No infinite velocity](#extreme-acceleration) `*`
- [No immediate acceleration](#extreme-acceleration) `*`
- [No self-intersections](#self-intersections) `*`
- No sub-frame keys `*`
- [Edge angles within -120 to 120 degrees on elastic surfaces](#extreme-surface-tangency) `*`
- [Edge lengths within 50-150% for elastic surfaces](#extreme-surface-stretch-or-compression) `*`
- [Edge lengths within 90-110% for rigid surfaces](#extreme-surface-stretch-or-compression) `*`

**Data**

- `label (str, optional)`: Pretty printed name in graphical user interfaces

**Sets**

- None

<br>

**Legend**

| Title               | Description
|:--------------------|:-----------
| **Target Audience** | Who is the end result of this family intended for?
| **Requirements**    | What is expected of this asset before it passes the tests?
| **Data**            | End-user configurable options
| **Sets**            | Collection of specific items for publishing or use further down the pipeline.

<br>
<br>

### Example

The following is an example of the minimal effort required to produce film with Starter and Autodesk Maya.

**Table of contents**

- [Setup](#setup)
- [Model](#model)
- [Rigging](#rigging)
- [Animation](#animation)

<br>

##### Setup

Before any work can be done, you must initialise Starter.

```python
# Prerequisite
import pyblish_maya
pyblish_maya.setup()

# Starter
import pyblish_starter
pyblish_starter.setup()
```

<br>

##### Modeling

Create a new model from scratch and publish it.

```python
from maya import cmds

cmds.file(new=True, force=True)

cmds.polyCube(name="Paul")
cmds.group(name="model")
instance = cmds.sets(name="Paul_model")

data = {
    "id": "pyblish.starter.instance",
    "family": "starter.model"
}

for key, value in data.items():
    cmds.addAttr(instance, longName=key, dataType="string")
    cmds.setAttr(instance + "." + key, value, type="string")

from pyblish import util
util.publish()
```

<br>

##### Rigging

Build upon the model from the previous example to produce a rig.

```python
import os
from maya import cmds
import pyblish_starter.maya
import pyblish_starter.maya.lib
reload(pyblish_starter.maya.lib)
reload(pyblish_starter.maya)
from pyblish_starter.maya import (
    hierarchy_from_string,
    outmesh,
    load
)

cmds.file(new=True, force=True)

# Load external asset
input_ = load("Paul_model", version=1, namespace="Paul_")
model_assembly = cmds.listRelatives(input_[0], children=True)[0]
model_geometry = outmesh(cmds.listRelatives(
    model_assembly, shapes=True)[0], name="Model")

assembly = hierarchy_from_string("""\
rig
    implementation
        input
        geometry
        skeleton
    interface
        controls
        preview
""")

# Rig
control = cmds.circle(name="Control")[0]
skeleton = cmds.joint(name="Skeleton")
preview = outmesh(model, name="Preview")
cmds.skinCluster(model, skeleton)
cmds.parentConstraint(control, skeleton)

# Sets
sets = list()
sets.append(cmds.sets(control, name="all_controls"))
sets.append(cmds.sets(model, name="all_cachable"))
sets.append(cmds.sets(reference, name="all_resources"))

# Organise
cmds.parent(input_, "input")
cmds.parent(control, "controls")
cmds.parent(skeleton, "skeleton")
cmds.parent(model, "geometry")
cmds.parent(preview, "interface|preview")
cmds.setAttr(control + ".overrideEnabled", True)
cmds.setAttr(control + ".overrideColor", 18)
cmds.hide("implementation")
cmds.select(deselect=True)

# Create instance
instance = cmds.sets([assembly] + sets, name="Paul_rig")

data = {
    "id": "pyblish.starter.instance",
    "family": "starter.rig"
}

for key, value in data.items():
    cmds.addAttr(instance, longName=key, dataType="string")
    cmds.setAttr(instance + "." + key, value, type="string")

from pyblish import util
#util.publish()
```

<br>

##### Animation

Build upon the previous example by referencing and producing an animation from the rig.

```python
from maya import cmds
from pyblish_starter.maya import (
    load
)

cmds.file(new=True, force=True)

# Load external asset
rig = load("Paul_rig", version=1, namespace="Paul01_")[0]

```

<br>
<br>

#### Requirements specification

The following is a details description of each requirement along with motivation and technical reasoning for their existence.

<br>

##### Workout

A workout is an animation clip associated with one or more character rigs. It contains both subtle and extreme poses along with corresponding transitions between them to thoroughly exercise the capabilities of a rig.

The workout is useful to both the character setup artist, the simulation artist and automated testing to visualise overall performance and behavior and to discover problems in unforeseen corner cases.

<br>

##### Self-intersections

Three dimensional geometric surfaces inherently share no concept of volume or mass, but both realism and subsequent physical simulations, such as clothing or hair, depend on it.

> Implementation tip: A toon shader provides an option to produce a nurbs curve or mesh from self-intersecting geometry. A plug-in could take advantage of this to test the existence of such a mesh either at standstill or in motion.

<br>

##### Extreme Acceleration

In reality, nothing is immediate. Even light takes time to travel from one point to another. For realism and post-processing of character animation, such as clothing and hair, care must be taken not to exceed realistic boundaries that may complicate the physical simulation of these materials.

<br>


##### Extreme Surface Tangency

In photo-realistic character animation, when the angle between two edges exceeds 120 degrees, an infinitely sharp angle appears that complicates life for artists relying on this surface for collisions.

To work around situations where the overall shape must exceed 120 degrees - such as in the elbow or back of a knee - use two or more edges. The sum of each edges contribute to well beyond 360 degrees and may be as short as is necessary.

<br>

##### Extreme Surface Stretch or Compression

Surface stretch and compression on elastic surfaces may negatively affect textures and overall realism.

<br>
<br>

#### Todo

Instances, in particular the Animation instance, requires some setup before being cachable. We don't want the user to perform this setup, but rather a tool. The tool could be in the form of a GUI that guides a user through selecting the appropriate nodes. Ideally the tools would be implicit in the loading of an asset through an asset library of sorts.

- Tool to create model, rig and animation instance.
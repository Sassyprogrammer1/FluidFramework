---
title: Property DataBinder Overview
menuPosition: 2
draft: true
---

We assume that you already have some basic knowledge of how to interact with the Property DDS and
its API. If not, we suggest to first read more about it on the
[Property DDS]({{< ref "start/property-dds/property-dds.md" >}}).

[Property DDS]({{< ref "start/property-dds/property-dds.md" >}}) is designed for writing collaborative, data-centric
applications. However, the mapping from the hierarchical data stored in
[Property DDS]({{< ref "start/property-dds/property-dds.md" >}}) to business logic, and processing of the changes to this
hierarchical data, is usually not straightforward. [Property DataBinder]({{< ref "docs/apis/property-binder" >}}) was
created to help map changes in the [Property DDS]({{< ref "start/property-dds/property-dds.md" >}}) hierarchy to
the runtime models in the application, and to abstract the low-level
changes to high-level objects and callback functions.

In the following, we will describe at a high level some of the primary features
of the [Property DataBinder]({{< ref "docs/apis/property-binder" >}}).

## Application Overview

![Property DataBinder Overview](/images/property_binder_overview.png)


## Property DDS and Fluid Document

In the above figure, the application has a Property DDS that is a
materialized view of the property data stored in a Fluid Document.
Changes to the Property DDS may be coming in from the Fluid Document,
from collaborators, or from local edits done by the
application.

### Runtime Representation

On the right, the application requires one or more sets of runtime
representations to represent the properties from the Property DDS These runtime
representations may be THREE.js graphics objects, UI Widgets in React or Angular, sketching
constraints, mesh faces or even simply sets of helpers to modify the
underlying Property DDS data.

A single Property may lead to multiple runtime representations; for
example, a Material may be mapped to a THREE.js material, but also to a
widget for modifying the material properties. In the figure, there are
two sets, one named ‘VIEW' and the other ‘MODEL'. These names (referred
to as `BindingTypes`) are chosen by the application and/or the component
defining the runtime models.


The [Property DataBinder]({{< ref "docs/apis/property-binder" >}}) provides functionality for managing these
representations, described separately in [Runtime Representations]({{< ref "property-binder-runtime-representation.md" >}})


## Property DataBindings

The update of the runtime representations, based on the changes
occurring to the Property DDS, are managed by data binding
classes. Applications create data binding classes that inherit from the
``DataBinding`` base class, and register them with the [DataBinder]({{< ref "docs/apis/property-binder/databinder" >}}) with
an Property Schema id, such as ``Material-1.0.0`` or ``Camera-1.0.0``.

When *Properties* of the corresponding *Schema* are created in the
Property DDS, the [DataBinder]({{< ref "docs/apis/property-binder" >}}) will instantiate the most
appropriate DataBinding class (and runtime representation) for that
type, taking inheritance into account. For example, if there is a
specialized *DataBinding* for the *Schema* ``PerspectiveCamera-1.0.0``,
inheriting from ``Camera-1.0.0``, the [DataBinder]({{< ref "docs/apis/property-binder/databinder" >}}) will instantiate it
if appropriate. When Properties corresponding to DataBindings are
removed from the Property DDS, [DataBinder]({{< ref "docs/apis/property-binder/databinder" >}}) will remove these DataBindings.

As with the runtime representations, the application-defined
`bindingType` can be used to create classes of DataBindings for
working with different representations. In the figure, the names ‘VIEW'
and ‘MODEL' are used.

## Callbacks

A `DataBinding` class, when instantiated, can overload various callbacks
to track the life-cycle of the associated Property. These are detailed
in [Property DataBinding Callbacks]({{< ref "property-binder-callbacks.md" >}}), but they include the insertion,
modification and removal of the corresponding Properties. In addition,
the `DataBinding` class can register for insertions, modifications and
removals of Properties at paths relative to the Property at which
the DataBinding class was instantiated - this allows the application
to react to specific changes in the data.

Callbacks can also be bound to absolute paths within the Property DDS (e.g.
to track changes to a specific path regardless of the type of Property
present at that path), these callbacks are independent of Property DataBindings.

Callbacks may be set up to track changes to collections (for example
inserts & removals from an array or map). References are also supported and
changes to both the reference *Property* itself and the referenced *Property*
can be tracked.

## Example

The following is a simple example of the usage of the [DataBinder]({{< ref "docs/apis/property-binder/databinder" >}})
to track changes to a graphics scene described with a Scene, Meshes, and Materials. For simplicity, we will focus on the
changes to the Material.

The *Schema* is defined as follows; the Scene contains a set of meshes, that have associated material properties. The
material color property is described as a string:

```javascript

  const sceneSchema = {
    typeid: 'Sample:Scene-1.0.0',
    properties: [
      {id: 'meshes', typeid: 'Sample:Mesh-1.0.0', context: 'set'}
    ]
  };
  const meshSchema = {
    typeid: 'Sample:Mesh-1.0.0',
    inherits: 'NamedProperty',
    properties: [
      {id: 'material', typeid: 'Sample:Material-1.0.0'}
    ]
  };
  const materialSchema = {
    typeid: 'Sample:Material-1.0.0',
    properties: [
      {id: 'color', typeid: 'String'}
    ]
  };
```

We populate our *Property DDS* with a Scene, containing a single mesh. We set the starting color of the Material to red:

```javascript

  const scene = PropertyFactory.create('Sample:Scene-1.0.0');
  const mesh = PropertyFactory.create('Sample:Mesh-1.0.0');
  // Insert a scene, containing a single mesh
  scene.get('meshes').insert(mesh);
  this._propertyDds.root.insert('scene', scene);
  // Set the mesh material to red
  mesh.get('material').setValue('rgb(255,0,0)');
```

We define a runtime representation for the material, which in this example, will be a
`THREE.MeshPhongMaterial`. We choose the string ‘VIEW' as the *bindingType*; this string could be any string we want,
to permit differentiating different runtime representations.

```javascript
myDataBinder.defineRepresentation('VIEW', 'Sample:Material-1.0.0', () => new THREE.MeshPhongMaterial());
```

Next, we define a class that extends the *DataBinding* class, to capture changes to the *Property* hieararchy. The
class can overload the constructor and the onPreRemove functions to capture changes to the lifetime of the
Material *Property*.

In the ``initialize()`` function, the class defines a callback to be notified if the subproperty ``color`` is inserted
(which happens on Material creation), or modified. Whenever the underlying Property DDS data is modified,
``changeColor`` is called and the data binding will update the `THREE.MeshPhongMaterial`.

The constructor is fetching the representation of the material using [getRepresentation()]({{< ref
"docs/apis/property-binder/databinding#property-binder-databinding-getrepresentation-Method" >}}). By default,
[getRepresentation()]({{< ref
"docs/apis/property-binder/databinding#property-binder-databinding-getrepresentation-Method" >}}) returns the
representation that has the same binding type as the instance of the *DataBinding*, which in this example is ‘VIEW'.

```javascript
  class MaterialDataBinding extends DataBinding {
    constructor(in_params) {
      super(in_params);
      this._material = this.getRepresentation();
    }
    changeColor(value) {
      this._material.color.set(value); // Will convert the string to r,g,b values
    }
    static initialize() {
      this.registerOnValues('color', ['modify', 'insert'], this.changeColor);
    }
  }

  MaterialDataBinding.initialize();
```


Finally, the application registers this DataBinding with the [DataBinder]({{< ref "docs/apis/property-binder/databinder"
>}}) by using [defineDataBinding()]({{< ref
"docs/apis/property-binder/databinder#property-binder-databinder-definedatabinding-Method" >}}), and then activates it.
By activating it, the [DataBinder]({{< ref "docs/apis/property-binder/databinder" >}}) will look for any
``Sample:Material-1.0.0`` Properties in the Property DDS, and instantiate the data bindings. The runtime representations
will be instantiated on demand when the [getRepresentation()]({{< ref
"docs/apis/property-binder/databinder#property-binder-databinder-getrepresentation-Method" >}}) call is made.

```javascript
  dataBinder.defineDataBinding('VIEW', 'Sample:Material-1.0.0', MaterialDataBinding);
  dataBinder.activateDataBinding('VIEW', 'Sample:Material-1.0.0');
```

Although we are defining the *DataBinding* and then immediately activating it, the expected usage is that the
component providing the functionality (such as graphics rendering) would define the runtime representations and
data bindings, while the application using the component would activate the bindings.

## Other Resources

* [Property DataBinder API]({{< ref "docs/apis/property-binder" >}})
* [Property DataBinder Binding Definition and Activation]({{< ref "property-binder-binding-definition.md" >}})
* [Property DataBinder Callbacks]({{< ref "property-binder-callbacks.md" >}})
* [Property DataBinder Runtime Representations]({{< ref "property-binder-runtime-representation.md" >}})
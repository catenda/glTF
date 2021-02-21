<!-- omit in toc -->
# EXT_feature_metadata

**Version 1.0.0** [TODO: date]

<!-- omit in toc -->
## Contributors

* Peter Gagliardi, Cesium
* Sean Lilley, Cesium
* Bao Tran, Cesium
* Sam Suhag, Cesium
* Samuel Vargas, Cesium
* Patrick Cozzi, Cesium

<!-- omit in toc -->
## Status

Draft

<!-- omit in toc -->
## Dependencies

Written against the glTF 2.0 specification.

Optionally extends the [`EXT_mesh_gpu_instancing` extension](../../EXT_mesh_gpu_instancing/README.md).

<!-- omit in toc -->
## Optional vs. Required

This extension is optional, meaning it should be placed in the `extensionsUsed` list, but not in the `extensionsRequired` list.

<!-- omit in toc -->
## Contents

- [Overview](#overview)
- [Feature Identification](#feature-identification)
  - [Feature ID Vertex Attributes](#feature-id-vertex-attributes)
    - [Feature ID Accessors](#feature-id-accessors)
    - [Implicit Feature IDs](#implicit-feature-ids)
  - [Feature ID Textures](#feature-id-textures)
  - [Feature ID Instance Attributes](#feature-id-instance-attributes)
- [Feature Metadata](#feature-metadata)
  - [Schemas](#schemas)
  - [Feature Tables](#feature-tables)
  - [Feature Textures](#feature-textures)
  - [Statistics](#statistics)
- [Examples](#examples)

## Overview

This extension allows offline batching of heterogeneous 3D features, such as different buildings in a city, for efficient streaming to a client for rendering and interaction. Efficiency comes from transferring multiple features in a single request and rendering them in the least number of draw calls necessary.

Feature IDs enable individual features to be identified and updated at runtime, e.g., show/hide, highlight color, etc. Feature IDs may be assigned on a per-vertex, per-texel, or per-instance basis.

Feature IDs may be used to access metadata, such as passing a building's ID to get its address. Per-feature metadata is stored in a compact binary tabular format described in the [Cesium 3D Metadata Specification](https://github.com/CesiumGS/3d-tiles/blob/3d-tiles-next/specification/Metadata/README.md).

![Building Example](figures/feature-table-buildings.png)

> In the diagram above, a glTF consists of two houses batched together into a single primitive. A feature ID attribute on the primitive indicates that all of the vertices making up the first house have a feature ID of 0, while all vertices making up the second house have the feature ID 1. The feature ID is then used to access the building's metadata from the feature table.

This extension also defines an alternate form of metadata storage that uses textures to store values directly. This format is especially useful when texture mapping high frequency data, like material properties, to less detailed 3D surfaces. Metadata textures enable new styling and analytics capabilities, and complement glTF PBR textures.

See [Use Cases](#use-cases) for a full list of use cases for this extension.

## Feature Identification

Features in a glTF primitive are identified in three ways:

* Per-vertex using a vertex attribute
* Per-texel using a glTF texture
* Per-instance using an instance attribute with the [`EXT_mesh_gpu_instancing` extension](../../EXT_mesh_gpu_instancing/README.md)

<img src="figures/metadata-access.png"  alt="Metadata Access" width="600">

### Feature ID Vertex Attributes

#### Feature ID Accessors

The most straightforward method for defining feature IDs is to store them in a glTF vertex attribute. Feature ID attributes must follow the naming convention `_FEATURE_ID_X` where `X` is a non-negative integer. The first feature ID attribute is `_FEATURE_ID_0`, the second `_FEATURE_ID_1`, and so on.

Feature IDs must be whole numbers in the range `[0, count - 1]`, where `count` is the total number of features in the feature table.

The attribute's accessor `type` must be `"SCALAR"` and `normalized` must be false. There is no restriction on `componentType`.

> **Implementation Note:** since glTF accessors do not support `UNSIGNED_INT` types for 32-bit integers, `FLOAT` may be used instead. This allows for integer feature IDs up to 2²⁴. For smaller ranges of feature IDs, `UNSIGNED_BYTE` or `UNSIGNED_SHORT` can still be used. Note that this requires aligning each feature ID to 4-byte boundaries to adhere to glTF's alignment rules.

![Feature Table](figures/feature-table.png)

```jsonc
{
  "primitives": [
    {
      "attributes": {
        "POSITION": 0,
        "_FEATURE_ID_0": 1
      },
      "indices": 2,
      "mode": 4,
      "extensions": {
        "EXT_feature_metadata": {
          "featureIdAttributes": [
            {
              "featureTable": "buildings",
              "featureIds": {
                "attribute": "_FEATURE_ID_0"
              }
            }
          ]
        }
      }
    }
  ]
}
```

#### Implicit Feature IDs

In some cases it is possible to define feature IDs implicitly, such as when all vertices in a primitive have the same feature ID or when each vertex in a primitive has a different feature ID.

This is accomplished by using `constant` and `divisor`.

* `constant` sets a constant feature ID for each vertex. The default is `0`.
* `divisor` sets the rate at which feature IDs increment. If `divisor` is zero then `constant` is used. If `divisor` is greater than zero the feature ID increments once per `divisor` sets of vertices, starting at `constant`. The default is `0`.

For example, if `constant` is 0 and `divisor` is 1 the feature IDs are `[0, 1, 2, 3, 4, 5, ...]`. If `constant` is 2 and `divisor` is 3 the feature IDs are `[2, 2, 2, 3, 3, 3, 4, 4, 4, ...]`.

`constant` and `divisor` must be omitted when `attribute` is used. These two methods of assigning feature IDs are mutually exclusive.

<img src="figures/placemarks.png"  alt="Placemarks" width="600">

```jsonc
{
  "primitives": [
    {
      "attributes": {
        "POSITION": 0
      },
      "mode": 0,
      "extensions": {
        "EXT_feature_metadata": {
          "featureIdAttributes": [
            {
              "featureTable": "placemarks",
              "featureIds": {
                "constant": 0,
                "divisor": 1
              }
            }
          ]
        }
      }
    }
  ]
}
```
### Feature ID Textures

Sometimes it is helpful to classify the pixels of an image into different features. Some examples include segmenting an image in a computer vision project or marking regions on a map.

Often per-texel feature IDs provide finer granularity than per-vertex feature IDs like in the example below.

<img src="figures/feature-id-texture.png"  alt="Feature ID Texture" width="800">

```jsonc
{
  "primitives": [
    {
      "attributes": {
        "POSITION": 0,
        "TEXCOORD_0": 1
      },
      "indices": 2,
      "material": 0,
      "extensions": {
        "EXT_feature_metadata": {
          "featureIdTextures": [
            {
              "featureTable": "buildingFeatures",
              "featureIds": {
                "texture": {
                  "texCoord": 0,
                  "index": 0
                },
                "channels": "r"
              }
            }
          ]
        }
      }
    }
  ]
}
```

`texture` is a glTF [`textureInfo`](../../../../../specification/2.0/schema/textureInfo.schema.json) object. `channels` must be a single channel (`"r"`, `"g"`, `"b"`, or `"a"`). Furthermore, feature IDs must be whole numbers in the range `[0, count - 1]`, where `count` is the total number of features in the feature table. Texture filtering should be disabled when accessing feature IDs.

### Feature ID Instance Attributes

Feature IDs may also be assigned to individual instances when using the [`EXT_mesh_gpu_instancing` extension](../../EXT_mesh_gpu_instancing/README.md). This works the same way as assigning feature IDs to vertices. Feature IDs may be stored in accessors or generated implicitly.

```jsonc
{
  "nodes": [
    {
      "mesh": 0,
      "extensions": {
        "EXT_mesh_gpu_instancing": {
          "attributes": {
            "TRANSLATION": 0,
            "ROTATION": 1,
            "SCALE": 2,
            "_FEATURE_ID_0": 
          },
          "extensions": {
            "EXT_feature_metadata": {
              "featureIdAttributes": [
                {
                  "featureTable": "trees",
                  "featureIds": {
                    "attribute": "_FEATURE_ID_0"
                  }
                }
              ]
            }
          }
        }
      }
    }
  ]
}
```

## Feature Metadata

Feature metadata is structured according to a **schema**. A schema has a set of **classes** and **enums**. A class contains a set of **properties**, which may be numeric, boolean, string, enum, or array types.

A **feature** is a specific instantiation of class containing **property values**. Property values are stored in either a **feature table** or a **feature texture** depending on the use case. Both formats are designed for storing property values for a large number of features.

**Statistics** provide aggregate information about the metadata. For example, statistics may include the min/max values of a numeric property for mapping property values to color ramps or the number of enum occurrences for creating histograms.

By default properties do not have any inherent meaning. A property may be assigned a **semantic**, an identifier that describes how the property should be interpreted. The full list of built-in semantics can be found in the [Cesium Metadata Semantics Reference](https://github.com/CesiumGS/3d-tiles/blob/3d-tiles-next/specification/Metadata/Semantics/README.md). Model authors may define their own application or domain-specific semantics separately.

This extension references the [Cesium 3D Metadata Specification](https://github.com/CesiumGS/3d-tiles/blob/3d-tiles-next/specification/Metadata/README.md), which describes the metadata format in full detail.

### Schemas

A schema defines a set of classes and enums used in a model. Classes serve as templates for entities - they provide a list of properties and the type information for those properties. Enums define the allowable values for enum properties. Schemas are defined in full detail in the [Schemas](https://github.com/CesiumGS/3d-tiles/tree/3d-tiles-next/specification/Metadata/1.0.0#schemas) section of the [Cesium 3D Metadata Specification](https://github.com/CesiumGS/3d-tiles/blob/3d-tiles-next/specification/Metadata/README.md).

A schema may be embedded in the extension directly or referenced externally with the `schemaUri` property. Multiple glTF models may refer to the same external schema to avoid duplication.

Schemas may be given a `name`, `description`, and `version`.

### Feature Tables

A feature table stores property values in a parallel array format. Each property array corresponds to a class property. The values contained within a property array must match the data type of the class property. Furthermore, the set of property arrays must match one-to-one with the class properties.

The schema and feature tables are defined in the root extension object in the glTF model. See the example below:

```jsonc
{
  "extensions": {
    "EXT_feature_metadata": {
      "schema": {
        "classes": {
          "tree": {
            "properties": {
              "height": {
                "name": "Height (m)",
                "description": "Height of tree measured from ground level",
                "type": "FLOAT32"
              },
              "birdCount": {
                "name": "Bird Count",
                "description": "Number of birds perching on the tree",
                "type": "UINT8"
              },
              "species": {
                "name": "Species",
                "description": "Species of the tree",
                "type": "STRING"
              }
            }
          }
        }
      },
      "featureTables": {
        "trees": {
          "class": "tree",
          "count": 10,
          "properties": {
            "height": {
              "bufferView": 0
            },
            "birdCount": {
              "bufferView": 1
            },
            "species": {
              "bufferView": 2,
              "arrayOffsetBufferView": 3,
              "stringOffsetBufferView": 4
            }
          }
        }
      }
    }
  }
}
```

`class` is the ID of the class in the schema. `count` is the number of features in the feature table, and the length of each property array. Property arrays are stored in glTF buffer views and use the binary encoding defined in the [Table Format](https://github.com/CesiumGS/3d-tiles/tree/3d-tiles-next/specification/Metadata/1.0.0#table-format) section of the [Cesium 3D Metadata Specification](https://github.com/CesiumGS/3d-tiles/blob/3d-tiles-next/specification/Metadata/README.md).

Each buffer view `byteOffset` must be aligned to a multiple of 8 bytes. For a GLB file, this is measured relative to the beginning of the file. For a glTF + BIN file, this is relative to the beginning of the BIN file.

### Feature Textures

Feature textures (not to be confused with [Feature ID Textures](#feature-id-texture)) are an alternate form of metadata storage that uses textures rather than parallel arrays to store values. Feature textures are accessed directly by texture coordinates, rather than feature IDs. Feature textures are especially useful when texture mapping high frequency data to less detailed 3D surfaces.

Feature textures use the [Raster Format](https://github.com/CesiumGS/3d-tiles/tree/3d-tiles-next/specification/Metadata/1.0.0#raster-format) of the [Cesium 3D Metadata Specification](https://github.com/CesiumGS/3d-tiles/blob/3d-tiles-next/specification/Metadata/README.md) with a few additional constraints:

* A scalar property cannot be encoded into multiple channels. For example, it is not possible to encode a `UINT32` property in an `RGBA8` texture.
* Components of fixed-length array properties must be separate channels within the same texture.
* Variable-length arrays are not supported

Additionally, the data type and bit depth of the image must be compatible with the property type. An 8-bit per pixel RGB image is only compatible with `UINT8` or normalized `UINT8` properties, and array properties thereof with three components or less. Likewise, a floating point property requires a floating point-compatible image format like KTX2 which may require additional extensions.

Feature textures are defined with the following steps:

* A class is defined in the root `EXT_feature_metadata` extension object. This is used to describe the metadata in the texture.
* A feature texture is defined in the root `EXT_feature_metadata.featureTextures` object. This must reference the class ID defined in step 1.
* A feature texture is associated with a primitive by listing the feature texture ID in the `primitive.EXT_feature_metadata.featureTextures` array.
<img src="figures/feature-texture.png"  alt="Feature Texture" width="500">

_Class and feature texture_

```jsonc
{
  "extensions": {
    "EXT_feature_metadata": {
      "schema": {
        "classes": {
          "heatLossSample": {
            "properties": {
              "heatLoss": {
                "type": "UINT8",
                "normalized": true
              }
            }
          }
        }
      },
      "featureTextures": {
        "heatLossTexture": {
          "class": "heatLossSample",
          "properties": {
            "heatLoss": {
              "texture": {
                "index": 0,
                "texCoord": 0
              },
              "channels": "g"
            }
          }
        }
      }
    }
  }
}
```

_Primitive association_

```jsonc
{
  "primitives": [
    {
      "attributes": {
        "POSITION": 0,
        "TEXCOORD_0": 1
      },
      "indices": 2,
      "material": 0,
      "extensions": {
        "EXT_feature_metadata": {
          "featureTextures": ["heatLossTexture"]
        }
      }
    }
  ]
}
```


`texture` is a glTF [`textureInfo`](../../../../../specification/2.0/schema/textureInfo.schema.json) object. `texCoord` refers to the texture coordinate set in the referring primitive. `channels` is a string matching the pattern `"^[rgba]{1,4}$"` that specifies which texture channels store property values.

### Statistics

Statistics provide aggregate information about features in the model. Statistics are provided on a per-class basis.

* `count` is the number of features that conform to the class
* `properties` contains statistics about property values

The following built-in statistics can be used:

Name|Description|Type
--|--|--
`min`|Minimum value|Numeric types or fixed-length arrays of numeric types
`max`|Maximum value|...
`mean`|The arithmetic mean of the values|...
`median`|The median value|...
`standardDeviation`|The standard deviation of the values|...
`variance`|The variance of the values|...
`sum`|The sum of the values|...
`occurrences`|Number of enum occurrences|Enums or fixed-length arrays of enums

Model authors may define their own additional statistics, like `mode` below.

```jsonc
{
  "extensions": {
    "3DTILES_metadata": {
      "schema": {
        "enums": {
          "buildingType": {
            "valueType": "UINT16",
            "values": [
              {
                "name": "Residential",
                "value": 0
              },
              {
                "name": "Commercial",
                "value": 1
              },
              {
                "name": "Hospital",
                "value": 2
              },
              {
                "name": "Other",
                "value": 3
              }
            ]
          }
        },
        "classes": {
          "building": {
            "properties": {
              "height": {
                "type": "FLOAT32"
              },
              "owners": {
                "type": "ARRAY",
                "componentType": "STRING"
              },
              "buildingType": {
                "type": "ENUM",
                "enumType": "buildingType"
              }
            }
          }
        }
      },
      "statistics": {
        "classes": {
          "building": {
            "count": 100,
            "properties": {
              "height": {
                "min": 3.9,
                "max": 341.7,
                "mode": 5.0
              },
              "buildingType": {
                "occurrences": {
                  "Residential": 70,
                  "Commercial": 28,
                  "Hospital": 2
                }
              }
            }
          }
        }
      }
    }
  }
}
```

## Examples

This section gives more examples.

_This section is non normative_

This extension supports a wide number of use cases

Example|Description|Image
--|--|--
Simple example
Per-vertex metadata||Accuracy
Point features||Float64 properties, strings, enums, etc
Multi-point features|For a concrete example, consider a point cloud of the house, as in the diagram below. The point cloud might be segmented into different regions (roof, window, walls, door), each with properties such as a component name or the year the component was built. The point cloud could also be considered as individual LiDAR samples with an intensity and temperature value. Both types of metadata can be described with this extension.|![Multi-point features](figures/point-cloud-layers.png)
Triangle and quad features
Instance features
Multi-instance features
Node features
Material classification||![Material Classification](figures/material-classification.png)
Multiple texture layers
Composite
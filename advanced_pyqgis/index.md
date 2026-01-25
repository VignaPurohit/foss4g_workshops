# Advanced PyQGIS Scripting and Automation

This workshop demonstrates how to build **interactive, production-oriented GIS tools** using PyQGIS.  
Rather than writing isolated scripts, we progressively evolve a **single interactive map tool** across multiple checkpoints, introducing new PyQGIS concepts at each stage.

---

## PyQGIS Classes and Methods Covered

The following PyQGIS and Qt classes are used throughout the workshop. Links point to the official documentation.

### Core Interaction & GUI
- [`QgsMapTool`](https://qgis.org/pyqgis/3.40/gui/QgsMapTool.html)
- [`QgsMapCanvas.setMapTool()`](https://qgis.org/pyqgis/3.40/gui/QgsMapCanvas.html#qgis.gui.QgsMapCanvas.setMapTool)
- [`iface.mapCanvas()`](https://qgis.org/pyqgis/3.40/gui/QgisInterface.html#qgis.gui.QgisInterface.mapCanvas)

### Temporary Geometry & Visual Feedback
- [`QgsRubberBand`](https://qgis.org/pyqgis/3.40/gui/QgsRubberBand.html)
- [`QgsWkbTypes`](https://qgis.org/pyqgis/3.40/core/QgsWkbTypes.html)

### Geometry & Spatial Objects
- [`QgsGeometry`](https://qgis.org/pyqgis/3.40/core/QgsGeometry.html)
- [`QgsPointXY`](https://qgis.org/pyqgis/3.40/core/QgsPointXY.html)

### Raster Analysis
- [`QgsRasterLayer`](https://qgis.org/pyqgis/3.40/core/QgsRasterLayer.html)
- [`QgsRasterBandStats`](https://qgis.org/pyqgis/3.40/core/QgsRasterBandStats.html)

### Vector Data & Persistence
- [`QgsVectorLayer`](https://qgis.org/pyqgis/3.40/core/QgsVectorLayer.html)
- [`QgsFeature`](https://qgis.org/pyqgis/3.40/core/QgsFeature.html)
- [`QgsField`](https://qgis.org/pyqgis/3.40/core/QgsField.html)
- [`QgsProject`](https://qgis.org/pyqgis/3.40/core/QgsProject.html)

### Qt (User Interaction)
- [`Qt`](https://doc.qt.io/qt-5/qt.html)
- [`QColor`](https://doc.qt.io/qt-5/qcolor.html)
- [`QInputDialog`](https://doc.qt.io/qt-5/qinputdialog.html)

---

## CHECKPOINT 1 — I can listen to map clicks

### Purpose
Learn how to interact with the QGIS map canvas using PyQGIS by capturing mouse events.

Key concepts:
- Event-driven programming
- Map tools vs scripts
- Screen coordinates vs map coordinates

### Code

```
from qgis.gui import QgsMapTool
from PyQt5.QtCore import Qt

class ClickLogger(QgsMapTool):
    def canvasPressEvent(self, event):
        if event.button() == Qt.LeftButton:
            pt = self.toMapCoordinates(event.pos())
            print("Clicked at:", pt)

tool = ClickLogger(iface.mapCanvas())
iface.mapCanvas().setMapTool(tool)
```


## CHECKPOINT 2 — I can draw temporary points (RubberBand)

### Purpose
This checkpoint introduces **visual feedback** on the map canvas.  
We draw **temporary geometry** to show that user interaction is working, without creating any real GIS data.

Nothing is saved. Nothing becomes a layer.

### Concepts Introduced
- Temporary vs permanent geometry
- RubberBand as a canvas overlay
- Tool state stored in `__init__`

### PyQGIS Classes & Methods
- `QgsRubberBand`
- `QgsWkbTypes.PointGeometry`
- `addPoint()`
- `reset()`

### Code

```
from qgis.gui import QgsMapTool, QgsRubberBand
from qgis.core import QgsWkbTypes
from PyQt5.QtCore import Qt
from PyQt5.QtGui import QColor

class PointSketcher(QgsMapTool):
    def __init__(self, canvas):
        super().__init__(canvas)
        self.rb = QgsRubberBand(canvas, QgsWkbTypes.PointGeometry)
        self.rb.setColor(QColor(255, 0, 0))
        self.rb.setWidth(5)

    def canvasPressEvent(self, event):
        if event.button() == Qt.LeftButton:
            pt = self.toMapCoordinates(event.pos())
            self.rb.addPoint(pt)

tool = PointSketcher(iface.mapCanvas())
iface.mapCanvas().setMapTool(tool)
```

To remove all temporary points:

```
tool.rb.reset(QgsWkbTypes.PointGeometry)
```


## CHECKPOINT 3 — I can sketch a polygon properly

### Purpose
- Polygon geometry
- Left click to add vertices
- Right click to finish geometry
- RubberBand stretch during mouse movement

### PyQGIS Classes & Methods Used
- `QgsMapTool`
- `QgsRubberBand`
- `QgsWkbTypes.PolygonGeometry`
- `QgsGeometry.fromPolygonXY()`
- `canvasPressEvent()`
- `canvasMoveEvent()`

### Code

```
from qgis.gui import QgsMapTool, QgsRubberBand
from qgis.core import QgsWkbTypes, QgsGeometry
from PyQt5.QtCore import Qt
from PyQt5.QtGui import QColor

class PolygonSketcher(QgsMapTool):
    def __init__(self, canvas):
        super().__init__(canvas)
        self.canvas = canvas
        self.points = []

        self.rb = QgsRubberBand(canvas, QgsWkbTypes.PolygonGeometry)
        self.rb.setColor(QColor(255, 0, 0, 100))
        self.rb.setWidth(2)

    def canvasPressEvent(self, event):
        if event.button() == Qt.LeftButton:
            pt = self.toMapCoordinates(event.pos())
            self.points.append(pt)
            self.rb.addPoint(pt, True)

        elif event.button() == Qt.RightButton:
            if len(self.points) < 3:
                print("Need at least 3 points")
                return

            geom = QgsGeometry.fromPolygonXY([self.points])
            print("Polygon WKT:")
            print(geom.asWkt())

            self.points = []
            self.rb.reset(QgsWkbTypes.PolygonGeometry)

    def canvasMoveEvent(self, event):
        if not self.points:
            return

        pt = self.toMapCoordinates(event.pos())
        if self.rb.numberOfVertices() > len(self.points):
            self.rb.removeLastPoint()
        self.rb.addPoint(pt, True)

tool = PolygonSketcher(iface.mapCanvas())
iface.mapCanvas().setMapTool(tool)
```

## CHECKPOINT 4 — Inspect raster statistics

### Purpose
- Combine geometry with raster context
- Inspect raster values interactively
- No saving or persistence
- Fast, exploratory statistics only

### PyQGIS Classes & Methods Used
- `QgsMapTool`
- `QgsRubberBand`
- `QgsGeometry`
- `QgsRasterLayer`
- `QgsRasterBandStats`
- `bandStatistics()`
- `canvasPressEvent()`
- `canvasMoveEvent()`

### Code

```
from qgis.gui import QgsMapTool, QgsRubberBand
from qgis.core import (
    QgsWkbTypes, QgsGeometry,
    QgsRasterLayer, QgsRasterBandStats
)
from PyQt5.QtCore import Qt
from PyQt5.QtGui import QColor

class RasterInspector(QgsMapTool):
    def __init__(self, canvas):
        super().__init__(canvas)
        self.canvas = canvas
        self.points = []

        self.rb = QgsRubberBand(canvas, QgsWkbTypes.PolygonGeometry)
        self.rb.setColor(QColor(255, 0, 0, 100))
        self.rb.setWidth(2)

    def canvasPressEvent(self, event):
        if event.button() == Qt.LeftButton:
            pt = self.toMapCoordinates(event.pos())
            self.points.append(pt)
            self.rb.addPoint(pt, True)

        elif event.button() == Qt.RightButton:
            if len(self.points) < 3:
                return

            geom = QgsGeometry.fromPolygonXY([self.points])
            self.inspect(geom)

            self.points = []
            self.rb.reset(QgsWkbTypes.PolygonGeometry)

    def canvasMoveEvent(self, event):
        if not self.points:
            return

        pt = self.toMapCoordinates(event.pos())
        if self.rb.numberOfVertices() > len(self.points):
            self.rb.removeLastPoint()
        self.rb.addPoint(pt, True)

    def inspect(self, geom):
        layer = self.canvas.currentLayer()
        if not isinstance(layer, QgsRasterLayer):
            print("Select a raster layer")
            return

        stats = layer.dataProvider().bandStatistics(
            1,
            QgsRasterBandStats.Mean |
            QgsRasterBandStats.Min |
            QgsRasterBandStats.Max,
            geom.boundingBox(),
            0
        )

        print("Mean:", stats.mean)
        print("Min :", stats.minimumValue)
        print("Max :", stats.maximumValue)

tool = RasterInspector(iface.mapCanvas())
iface.mapCanvas().setMapTool(tool)
```

## CHECKPOINT 5 — Inspection + Pending State  
*(Inspect ≠ Save)*

### Purpose
- Introduce a **pending geometry** state
- Show raster statistics before saving
- Require explicit user decision
- Allow discard using `ESC`

### What This Step Introduces
- Pending geometry and statistics
- Human-in-the-loop inspection
- Tool state management
- Keyboard interaction (`ESC`)

### PyQGIS Classes & Methods Used
- `QgsMapTool`
- `QgsRubberBand`
- `QgsGeometry`
- `QgsRasterLayer`
- `QgsRasterBandStats`
- `activate()`
- `keyPressEvent()`
- `bandStatistics()`

### Code

```
from qgis.gui import QgsMapTool, QgsRubberBand
from qgis.core import (
    QgsWkbTypes,
    QgsGeometry,
    QgsRasterLayer,
    QgsRasterBandStats
)
from PyQt5.QtCore import Qt
from PyQt5.QtGui import QColor

class RasterInspectorPending(QgsMapTool):
    def __init__(self, canvas):
        super().__init__(canvas)
        self.canvas = canvas

        # drawing state
        self.points = []

        # pending state
        self.pending_geom = None
        self.pending_stats = None
        self.is_pending = False

        self.rb = QgsRubberBand(canvas, QgsWkbTypes.PolygonGeometry)
        self.rb.setColor(QColor(255, 0, 0, 120))  # red = drawing
        self.rb.setWidth(2)

    def activate(self):
        self.canvas.setFocusPolicy(Qt.StrongFocus)
        self.canvas.setFocus()
        super().activate()

    def canvasPressEvent(self, event):
        if self.is_pending:
            print("Decision pending. Press ESC to discard.")
            return

        if event.button() == Qt.LeftButton:
            pt = self.toMapCoordinates(event.pos())
            self.points.append(pt)
            self.rb.addPoint(pt, True)

        elif event.button() == Qt.RightButton:
            if len(self.points) < 3:
                print("Need at least 3 points")
                return

            geom = QgsGeometry.fromPolygonXY([self.points])
            stats = self.inspect_raster(geom)

            if not stats:
                return

            self.pending_geom = geom
            self.pending_stats = stats
            self.is_pending = True

            self.rb.setColor(QColor(0, 180, 0, 120))  # green = pending

            print("---- Decision ----")
            print("This polygon is PENDING.")
            print("Press ESC to discard.")
            print("------------------")

            self.points = []

    def canvasMoveEvent(self, event):
        if not self.points or self.is_pending:
            return

        pt = self.toMapCoordinates(event.pos())
        if self.rb.numberOfVertices() > len(self.points):
            self.rb.removeLastPoint()
        self.rb.addPoint(pt, True)

    def keyPressEvent(self, event):
        if event.key() == Qt.Key_Escape:
            print("Pending polygon discarded.")
            self._reset()

    def inspect_raster(self, geom):
        layer = self.canvas.currentLayer()
        if not isinstance(layer, QgsRasterLayer):
            print("Select a raster layer")
            return None

        stats = layer.dataProvider().bandStatistics(
            1,
            QgsRasterBandStats.Mean |
            QgsRasterBandStats.Min |
            QgsRasterBandStats.Max,
            geom.boundingBox(),
            0
        )

        print("---- Raster Statistics ----")
        print("Layer:", layer.name())
        print("Mean:", stats.mean)
        print("Min :", stats.minimumValue)
        print("Max :", stats.maximumValue)
        print("---------------------------")

        return {
            "raster": layer.name(),
            "mean": stats.mean,
            "min": stats.minimumValue,
            "max": stats.maximumValue
        }

    def _reset(self):
        self.points = []
        self.pending_geom = None
        self.pending_stats = None
        self.is_pending = False

        self.rb.reset(QgsWkbTypes.PolygonGeometry)
        self.rb.setColor(QColor(255, 0, 0, 120))


tool = RasterInspectorPending(iface.mapCanvas())
iface.mapCanvas().setMapTool(tool)
```

## CHECKPOINT 6 — Commit Pending Geometry as Data  
*(Decision → Persistence)*

### Purpose
- Convert inspected geometries into real GIS data
- Respect user decision before saving
- Persist geometry only on explicit confirmation
- Enable collection of samples / QA polygons

### What This Step Introduces
- Saving geometry using keyboard interaction (`S` key)
- Discarding geometry using `ESC`
- Lazy creation of a memory vector layer
- Attribute input at save time
- CRS-safe geometry storage

### PyQGIS Classes & Methods Used
- `QgsMapTool`
- `QgsRubberBand`
- `QgsGeometry`
- `QgsRasterLayer`
- `QgsRasterBandStats`
- `QgsProject`
- `QgsVectorLayer`
- `QgsFeature`
- `QgsField`
- `QInputDialog`
- `keyPressEvent()`

### Code

```
from qgis.gui import QgsMapTool, QgsRubberBand
from qgis.core import (
    QgsWkbTypes,
    QgsGeometry,
    QgsRasterLayer,
    QgsRasterBandStats,
    QgsProject,
    QgsVectorLayer,
    QgsFeature,
    QgsField
)
from PyQt5.QtCore import Qt, QVariant
from PyQt5.QtGui import QColor
from PyQt5.QtWidgets import QInputDialog


class RasterInspectorCommit(QgsMapTool):
    def __init__(self, canvas):
        super().__init__(canvas)
        self.canvas = canvas

        self.points = []
        self.pending_geom = None
        self.pending_stats = None
        self.is_pending = False

        self.rb = QgsRubberBand(canvas, QgsWkbTypes.PolygonGeometry)
        self.rb.setColor(QColor(255, 0, 0, 120))
        self.rb.setWidth(2)

    def activate(self):
        self.canvas.setFocusPolicy(Qt.StrongFocus)
        self.canvas.setFocus()
        super().activate()

    def canvasPressEvent(self, event):
        if self.is_pending:
            print("Decision pending. Press S to save or ESC to discard.")
            return

        if event.button() == Qt.LeftButton:
            pt = self.toMapCoordinates(event.pos())
            self.points.append(pt)
            self.rb.addPoint(pt, True)

        elif event.button() == Qt.RightButton:
            if len(self.points) < 3:
                return

            geom = QgsGeometry.fromPolygonXY([self.points])
            stats = self.inspect_raster(geom)
            if not stats:
                return

            self.pending_geom = geom
            self.pending_stats = stats
            self.is_pending = True

            self.rb.setColor(QColor(0, 180, 0, 120))

            print("---- Decision ----")
            print("Press 'S' to SAVE")
            print("Press 'ESC' to DISCARD")
            print("------------------")

            self.points = []

    def canvasMoveEvent(self, event):
        if not self.points or self.is_pending:
            return

        pt = self.toMapCoordinates(event.pos())
        if self.rb.numberOfVertices() > len(self.points):
            self.rb.removeLastPoint()
        self.rb.addPoint(pt, True)

    def keyPressEvent(self, event):
        if event.key() == Qt.Key_Escape:
            print("Pending polygon discarded.")
            self._reset()

        elif event.key() == Qt.Key_S:
            if not self.is_pending:
                print("Nothing to save.")
                return

            self.save_geometry(self.pending_geom, self.pending_stats)
            self._reset()

    def inspect_raster(self, geom):
        layer = self.canvas.currentLayer()
        if not isinstance(layer, QgsRasterLayer):
            print("Select a raster layer")
            return None

        stats = layer.dataProvider().bandStatistics(
            1,
            QgsRasterBandStats.Mean |
            QgsRasterBandStats.Min |
            QgsRasterBandStats.Max,
            geom.boundingBox(),
            0
        )

        print("---- Raster Statistics ----")
        print("Layer:", layer.name())
        print("Mean:", stats.mean)
        print("Min :", stats.minimumValue)
        print("Max :", stats.maximumValue)
        print("---------------------------")

        return {
            "raster": layer.name(),
            "mean": stats.mean,
            "min": stats.minimumValue,
            "max": stats.maximumValue
        }

    def save_geometry(self, geom, stats):
        layer_name = "inspection_samples"
        project = QgsProject.instance()

        layers = project.mapLayersByName(layer_name)
        if layers:
            vlayer = layers[0]
        else:
            crs_authid = self.canvas.mapSettings().destinationCrs().authid()
            vlayer = QgsVectorLayer(
                f"Polygon?crs={crs_authid}",
                layer_name,
                "memory"
            )
            provider = vlayer.dataProvider()
            provider.addAttributes([
                QgsField("sample_id", QVariant.String),
                QgsField("note", QVariant.String),
                QgsField("raster", QVariant.String),
                QgsField("mean", QVariant.Double),
                QgsField("min", QVariant.Double),
                QgsField("max", QVariant.Double)
            ])
            vlayer.updateFields()
            project.addMapLayer(vlayer)

        sample_id, ok1 = QInputDialog.getText(None, "Sample ID", "Enter sample ID:")
        if not ok1:
            return

        note, ok2 = QInputDialog.getText(None, "Note", "Enter note:")
        if not ok2:
            return

        feat = QgsFeature(vlayer.fields())
        feat.setGeometry(geom)
        feat.setAttributes([
            sample_id,
            note,
            stats["raster"],
            stats["mean"],
            stats["min"],
            stats["max"]
        ])

        vlayer.dataProvider().addFeature(feat)
        vlayer.updateExtents()
        vlayer.triggerRepaint()

        print("✔ Geometry saved")

    def _reset(self):
        self.points = []
        self.pending_geom = None
        self.pending_stats = None
        self.is_pending = False

        self.rb.reset(QgsWkbTypes.PolygonGeometry)
        self.rb.setColor(QColor(255, 0, 0, 120))


tool = RasterInspectorCommit(iface.mapCanvas())
iface.mapCanvas().setMapTool(tool)
```

<script src="https://giscus.app/client.js"
        data-repo="username/foss4g_workshops"
        data-repo-id="REPO_ID"
        data-category="General"
        data-category-id="CATEGORY_ID"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-theme="light"
        crossorigin="anonymous"
        async>
</script>
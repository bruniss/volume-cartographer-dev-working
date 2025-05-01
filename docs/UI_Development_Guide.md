# Volume Cartographer UI Development Guide

This document provides information on common functions, methods, and classes that developers might need when making UI changes to the Volume Cartographer application.

## Table of Contents
1. [Overview](#overview)
2. [Key Classes](#key-classes)
3. [UI Architecture](#ui-architecture)
4. [Common Operations](#common-operations)
5. [Workflow Examples](#workflow-examples)

## Overview

Volume Cartographer is a Qt-based application for visualizing and manipulating volumetric data. The UI is built around several key components:

- **Volume Package Panel** - Shows available volume packages and segmentations
- **Viewer Windows** - Multiple MDI windows showing different views of the volumetric data
- **Layer Settings Panel** - Controls for manipulating layers and operations
- **Segmentation Panel** - Tools for segmentation operations

## Key Classes

### CWindow
The main application window class. It serves as the central controller for the application and manages:
- Loading/closing volume packages
- Creating and managing viewer windows
- Handling user interactions
- Managing UI state

Key methods:
- `OpenVolume(path)` - Opens a volume package
- `setVolume(newvol)` - Sets the current volume
- `onRefreshListPressed()` - Refreshes the volume package list
- `onSegFilterChanged(index)` - Updates the UI based on segmentation filter selection
- `onSurfaceSelected(current, previous)` - Handles surface selection in the tree view

### CVolumeViewer
Handles the display of volume slices and provides interaction with the visualized volume data.

Key methods:
- `OnVolumeChanged(vol)` - Updates the viewer when the volume changes
- `setSurface(name)` - Sets the surface to display
- `renderVisible(force)` - Renders the visible area
- `renderIntersections()` - Renders surface intersections
- `setIntersects(set)` - Sets which intersections to display
- `invalidateVis()` - Invalidates and forces a refresh of visualizations
- `invalidateIntersect(name)` - Invalidates intersection visualizations

### CSurfaceCollection
Manages collections of surfaces and their relationships.

Key methods:
- `setSurface(name, surface)` - Adds or updates a surface
- `removeSurface(name)` - Removes a surface
- `surface(name)` - Retrieves a surface by name
- `setPOI(name, poi)` - Sets a Point of Interest
- `poi(name)` - Retrieves a POI by name

### Surface Classes
- `PlaneSurface` - Represents a plane in 3D space
- `QuadSurface` - Represents a quad-based surface
- `OpChain` - A chain of operations applied to a surface

## UI Architecture

The UI follows a Model-View-Controller pattern:
- **Model**: Volume packages, surfaces, and POIs form the data model
- **View**: Various viewer widgets display the model
- **Controller**: CWindow and other interaction handlers

Signal/slot connections are extensively used for communication between components:
```cpp
// Example connections
connect(this, &CWindow::sendVolumeChanged, volView, &CVolumeViewer::OnVolumeChanged);
connect(_surf_col, &CSurfaceCollection::sendSurfaceChanged, volView, &CVolumeViewer::onSurfaceChanged);
connect(volView, &CVolumeViewer::sendVolumeClicked, this, &CWindow::onVolumeClicked);
```

## Common Operations

### Refreshing the Volume Package List
The volume package list can be refreshed to pick up new segmentations or changes:

```cpp
// Implementation from onRefreshListPressed()
// 1. Clear existing data
for (auto& pair : _vol_qsurfs) {
    _surf_col->removeSurface(pair.first);
}
_opchains.clear();
_vol_qsurfs.clear();

// 2. Reload segmentations
std::vector<std::string> seg_ids = fVpkg->segmentationIDs();
// Load segmentations in parallel
#pragma omp parallel for
for(int i=0; i<seg_ids.size(); i++) {
    // Process each segmentation...
}

// 3. Update UI
onSegFilterChanged(cmbFilterSegs->currentIndex());

// 4. Update viewers
for (auto &viewer : _viewers) {
    viewer->invalidateVis();
    viewer->invalidateIntersect();
    viewer->renderVisible(true);
    viewer->renderIntersections();
}
```

### Adding New UI Elements
To add new UI elements:

1. Edit the `VCMain.ui` file using Qt Designer or by directly modifying the XML
2. Add any required signals and slots in the appropriate class
3. Connect the UI element to its handler in `CreateWidgets()`

Example:
```cpp
// In CWindow::CreateWidgets()
QPushButton* myButton = this->findChild<QPushButton*>("myButtonName");
connect(myButton, &QPushButton::pressed, this, &CWindow::onMyButtonPressed);
```

### Working with Surfaces
Surfaces can be created, modified, and visualized:

```cpp
// Create a plane surface
PlaneSurface* plane = new PlaneSurface({x,y,z}, {nx,ny,nz});
_surf_col->setSurface("my_plane", plane);

// Update a surface
PlaneSurface* existingPlane = dynamic_cast<PlaneSurface*>(_surf_col->surface("my_plane"));
if (existingPlane) {
    existingPlane->setNormal({nx,ny,nz});
    _surf_col->setSurface("my_plane", existingPlane);
}

// Show intersections between surfaces
viewer->setIntersects({"my_plane", "another_surface"});
```

### Handling User Interaction
User interactions are processed through event handlers that modify the data model:

```cpp
// When a user clicks on a volume
void CWindow::onVolumeClicked(cv::Vec3f vol_loc, cv::Vec3f normal, Surface *surf, 
                             Qt::MouseButton buttons, Qt::KeyboardModifiers modifiers)
{
    // Handle various modifier keys and actions
    if (modifiers & Qt::ShiftModifier) {
        // Add points
    }
    else if (modifiers & Qt::ControlModifier) {
        // Update focus point
    }
    else {
        // Default action
    }
}
```

## Workflow Examples

### Adding a New Button to Refresh the List
1. Add the button to the UI file:
```xml
<widget class="QPushButton" name="btnRefreshList">
    <property name="text">
        <string>refresh list</string>
    </property>
</widget>
```

2. Add the slot to handle the button press:
```cpp
// In CWindow.hpp
public slots:
    void onRefreshListPressed();
```

3. Connect the button to the slot:
```cpp
// In CWindow::CreateWidgets()
connect(this->findChild<QPushButton*>("btnRefreshList"), &QPushButton::pressed, 
        this, &CWindow::onRefreshListPressed);
```

4. Implement the handler to refresh the list:
```cpp
void CWindow::onRefreshListPressed()
{
    // Implementation details...
}
```

### Updating Viewers After Data Changes
When data changes, viewers need to be updated:

```cpp
// Invalidate cached visualizations
viewer->invalidateVis();
viewer->invalidateIntersect();

// Force a complete redraw
viewer->renderVisible(true);
viewer->renderIntersections();
```

### Working with Points of Interest (POIs)
POIs are used to track specific locations in the volume:

```cpp
// Create or update a POI
POI *poi = _surf_col->poi("focus");
if (!poi) {
    poi = new POI;
}
poi->p = cv::Vec3f(x, y, z);
poi->n = cv::Vec3f(nx, ny, nz);
_surf_col->setPOI("focus", poi);

// Use POI information to update UI
lblLoc[0]->setText(QString::number(poi->p[0]));
lblLoc[1]->setText(QString::number(poi->p[1]));
lblLoc[2]->setText(QString::number(poi->p[2]));
```

This guide should help developers understand the key components and patterns used in the Volume Cartographer UI, making it easier to implement UI changes and extensions.

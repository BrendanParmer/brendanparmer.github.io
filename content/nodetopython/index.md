+++
date = '2026-03-15T14:50:13-05:00'
draft = false
title = 'Node To Python'
+++

![NodeToPython logo](/assets/img/node_to_python.png)

## Overview
Node To Python ([Github](https://github.com/BrendanParmer/NodeToPython), [Blender Extension Platform](https://extensions.blender.org/add-ons/node-to-python/)) is a Blender extension I created to turn node groups into Python scripts and standalone add-ons for recreating them. It supports all node tree types within Blender: geometry nodes, object materials, compositing nodes, and misc. shaders. Especially if you need custom functionality or to modify a node group dynamically, Node To Python is a great way to automatically get Python code for creating it.

The [README](https://github.com/BrendanParmer/NodeToPython/blob/main/docs/README.md) is a pretty good overview of how to use the project. The rest of this post will be more geared towards the development of NodeToPython, written as it stands in v4.1

## Project Structure
The codebase contains two directories of note:
1. `tools/` contains some scripts for helping with the development of NodeToPython. Since NodeToPython is a tool to help artists and programmers make tools, `tools/` contains the tools to make the tool that makes tools (I currently have no plans for recursing further)
    1. `node_settings_generator/parse_nodes.py` scrapes the Blender documentation to find out the structure of each node in each Blender version, compiling all nodes and settings into a dictionary to be used while exporting. Updating this dictionary was previously a manual and error-prone process, and now new Blender versions are much easier to integrate
    1. `package.py` is a simple script to generate the extension's `.zip`. I'd often forget to remove the pycache directories on release, so it's nice to have this automated now
2. `NodeToPython/` is the extension itself. It's divided into two parts:
    1. `ui/` contains the code for drawing the UI elements, selecting settings, and adding node groups for export. The code's not particularly interesting, but the UX has come a long way over the last four years. You used to have to push a button hidden away under the Object menu and type in the name of the node group you wanted to export. This is still evolving and ever-improving as I get feedback and suggestions from users
    2. `export/` is where the bulk of the logic resides. We'll dive in a little deeper in the next section

## How NodeToPython exports a node tree
* `ntp_operator.py` contains the actual function that gets called when hitting the export button. It'll check to make sure your Blender version is compatible with NodeToPython, register all the settings, handle file creation, and calculate the export order of the node groups
    * This last point is particularly important. Blender's node groups can be used within other node groups. That means we end up needing to find an order to export the node groups so that we only reference node groups that've already been created. For example, say we have node groups A, B, C, and D. Let A contain Group Nodes for B and C, B contain a Group Node for C, and D contain a Group Node for C, as well, such that we have a dependency graph:
        ```mermaid
        graph LR;
            A-->B;
            A-->C;
            B-->C;
            D-->C;
        ```
        We'll want to figure out which order is best to export the nodes in. Here, if we export groups A and D, we'll want to process C first, and then B before A, such that we don't duplicate work. A topological sort is a good fit here. The `_calculate_export_order()` method is designed to work for exporting potentially multiple node trees at a time, sorting dependencies into shared or specific modules as appropriate. If we again export A and D, node group C would go into the shared module, D into its own module, and B and A into A's module. If we removed D from our export list, then A, B, and C could all go into the same module. It also helps with handling node groups from the Blender Essentials library, which we don't want to duplicate if we don't have to.
        * If A, B, C, and D were geometry node groups, the generated code would look something like this. The created operator only calls functions to create the dependencies it needs.
            ```
            node_groups_to_export = [A, D]
            ```
            `geometry_common.py`:

            ```python3
            def c_node_group(node_tree_names: dict[typing.Callable, str]):
                c = bpy.data.node_groups.new(type='GeometryNodeTree', name="C")
                ...
                return c
            ```

            `d.py`:
            ```python3
            from . import x_common

            def d(node_tree_names: dict[typing.Callable, str]):
                d = bpy.data.node_groups.new(type='GeometryNodeTree', name="D")
                ...
                group = d.nodes.new("GeometryNodeGroup")
                group.node_tree = bpy.data.node_groups[node_tree_names[geometry_common.c_node_group]]
                ...
                return d
            
            class My_Add_on_OT_D(bpy.types.Operator):
                ...

                def execute(self, context: bpy.types.Context):
                    # Maps node tree creation functions to the node tree 
                    # name, such that we don't recreate node trees unnecessarily
                    node_tree_names : dict[typing.Callable, str] = {}

                    c = geometry_common.c_node_group(node_tree_names)
                    node_tree_names[geometry_common.c_node_group] = c.name

                    d = d_node_group(node_tree_names)
                    node_tree_names[d_node_group] = d.name

                    return {'FINISHED'}
            ```

            `a.py`:
            ```python3
            from . import geometry_common

            def b_node_group(node_tree_names: dict[typing.Callable, str]):
                b = bpy.data.node_groups.new(type='GeometryNodeTree', name="B")
                ...

                group = b.nodes.new("GeometryNodeGroup")
                group.node_tree = bpy.data.node_groups[node_tree_names[geometry_common.c_node_group]]
                ...
                return b

            def a_node_group(node_tree_names: dict[typing.Callable, str]):
                a = bpy.data.node_groups.new(type='GeometryNodeTree', name="A")
                ...
                group = a.nodes.new("GeometryNodeGroup")
                group.node_tree = bpy.data.node_groups[node_tree_names[b_node_group]]
                ...
                group_001 = a.nodes.new("GeometryNodeGroup")
                group_001.node_tree = bpy.data.node_groups[node_tree_names[geometry_common.c_node_group]]
                ...
                return a

            class My_Add_on_OT_A(bpy.types.Operator):
                ...

                def execute(self, context: bpy.types.Context):
                    # Maps node tree creation functions to the node tree 
                    # name, such that we don't recreate node trees unnecessarily
                    node_tree_names : dict[typing.Callable, str] = {}

                    c = geometry_common.c_node_group(node_tree_names)
                    node_tree_names[geometry_common.c_node_group] = c.name

                    b = b_node_group(node_tree_names)
                    node_tree_names[b_node_group] = b.name

                    a = a_node_group(node_tree_names)
                    node_tree_names[a_node_group] = a.name

                    return {'FINISHED'}
            ```
        

    * The operator will also keep track of the variable names we use for objects, to make sure there's no conflicts. To keep the generated script legible, our node variables try to stay close to the node's name, incrementing a counter if a variable with that name already exists
* `node_tree_exporter.py` contains the (abstract) base class for exporting a node tree. The operator will select the appropriate child exporter depending on the node tree's type (Compositor, Geometry, or Shader). Some particularly important things about this module:
    * `_create_obj()` is an optional method to create an object alongside the node tree. Materials, scenes, lights, line style, and worlds have other properties besides their node tree that we may wish to replicate, so it's important to have the ability to create them, as well. 
    * `_process_node_tree()` generates the Python code to actually recreate a node tree. Here, we set some properties about the node tree, defines its inputs and outputs, create all the nodes, and link them up
    * `_process_node()` generates the Python code for an individual node. Using the settings from `node_settings.py`, it'll replicate any properties the node had. It'll also set any default values for unlinked sockets
    * Zones are created with an input and output linked to each other. We keep track of the inputs so that we can later pair them with their corresponding output after all the nodes have been created
    * Some types have default values that depend on linked nodes, like menus for example. We'll create and store some lambdas to set those values after all nodes are linked

* `node_settings.py` is auto-generated from `/tools/node_settings_generator/parse_nodes.py`. This contains all the information we need about the different nodes and settings across multiple Blender versions that we wish to replicate
    * "Node settings" are the properties of a specific node that aren't set via sockets (can be linked to another node) or common to all node types (e.g. dimensions, color). For each node, we'll have a list of these, with their names, type, and min/max versions
    * Individual nodes can also have a min/max version. For now, this is mostly used as a sanity check
* `node_group_gatherer.py` grabs the node groups for export from the UI, and also calculates some stats about them
* `utils.py` provides some helper functions, mostly to do with type to string conversions for use in the generated code

## Developer Environment Setup and Practices
* I'm currently hosting the source code on [GitHub](https://github.com/BrendanParmer/NodeToPython). This is also where issue reports, task tracking, and code reviews are done for the project
* NodeToPython uses git for version control. Commits are typically prefixed with one of `feat`, `fix`, `cleanup`, `refactor`, or `docs`, but others can be used
* Personally, I use VSCode with Jacque Lucke's [Blender Development](https://marketplace.visualstudio.com/items?itemName=JacquesLucke.blender-development) extension. With the NodeToPython repo open, you can run the `Blender: Start` command from the Command Palette, from which you can select and launch an executable. In the extension settings you can give these executables nicer names than just paths. You can set breakpoints for debugging. It's also got a `Blender: Reload Addons` command that's pretty helpful
* For this project, I'm using a feature branching-ish strategy. `main` is considered stable and is what's been most recently published to the wider public. New features branch off and into a branch titled for the next minor release
* Versions are semantic, `{major}.{minor}.{subminor}`. Major releases tend to encompass big new features or refactors, such as UI revamps or support for new node tree types. Minor releases are for smaller features or new Blender versions. Subminor releases are for bug fixing only
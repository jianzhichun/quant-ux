<template>
     <div class="MatcToolbarLayerList MatcToobarPropertiesSection MatcToolbar">
		<div class="MatcToolbarLayerListCntr" data-dojo-attach-point="cntr">
			<div class="MatcToolbarLayerListScreenCntr">
				<div class="MatcLayerListScreens">
					<div :class="['MatcToolbarSection', {'MatcToolbarSectionCollabsed': isTreeCollapsed(tree)}]" v-for="tree in trees" :key="tree.id">
						<div class="MatcToolbarLayerListScreenLabel" @click="toggleTreeCollapsed(tree)">
							{{tree.name}}
							<span class="MatcToolbarSectionChevron MatcLayerListExpandIcon mdi mdi-chevron-down">
							</span>
						</div>
						<div class=" MatcToolbarSectionContent">
							<Tree
								:value="tree"
								@open="onOpen"
								@select="onSelect"
								@changeLabel="onChangeLabel"
								@dnd="onDnd"/>
						</div>
					</div>
				</div>
			</div>
		</div>
	</div>
</template>
<script>
import DojoWidget from 'dojo/DojoWidget'
import Logger from 'common/Logger'
import Util from 'core/Util'
import Tree from 'common/Tree'

export default {
	name: 'LayerList',
	props: ['value'],
    mixins:[Util, DojoWidget],
    data: function () {
        return {
            sections: [],
            screenListeners: {},
			collapsed: {},
			openNodes: {},
			trees: [],
			nodes: {}
        }
    },
    components: {
		'Tree': Tree
	},
    methods: {
      	postCreate (){
			this.logger = new Logger("LayerList");
			this.logger.log(-1,"constructor", "entry > " + this.mode);
		},

		setController (c){
			this.controller = c;
		},

		setCanvas (c){
			this.canvas = c;
		},

		setToolbar (t) {
			this.toolbar = t;
		},

		isTreeCollapsed (tree) {
			if (this.collapsed[tree.id] !== null && this.collapsed[tree.id] != undefined) {
				return this.collapsed[tree.id]
			}
			return false
		},

		toggleTreeCollapsed (tree) {
			if (this.collapsed[tree.id] !== null && this.collapsed[tree.id] != undefined) {
				this.$set(this.collapsed, tree.id, !this.collapsed[tree.id])
			} else {
				this.$set(this.collapsed, tree.id, true)
			}
		},

		onOpen (id, open) {
			this.logger.log(-1, "onOpen", "entry > ", id + ': ' + open);
			this.openNodes[id] = open
		},

		onSelect (ids) {
			this.logger.log(-1, "onSelect", "entry > ", ids);

			if (ids.length === 1) {
				let node = this.nodes[ids[0]]
				if (node) {
					if (this.canvas) {
						let type = node.type
						if (type === 'widget') {
							this.canvas.onWidgetSelected(node.id, true, true);
						}
						if (type === 'group') {
							this.canvas.onGroupSelected(node.id, true);
						}
					}
				} else {
					this.logger.log(-1, "onSelect", "No node > ", ids);
				}
			} else {
				if (this.canvas) {
					this.canvas.onMutliSelected(ids, true);
				}
			}
		},

		onChangeLabel (id, txt) {
			this.logger.log(1, "onChangeLabel", "entry > ", id + ': ' + txt);
			if (this.toolbar && this.controller) {
				let node = this.nodes[id]
				if (node) {
					this.toolbar.onModelNameChange(id, node.type, txt);
					let type = node.type
					if (type == "widget"){
						this.controller.setWidgetName(id, txt)
					} else if (type == "screen"){
						this.controller.setScreenName(id, txt)
					} else if (type == "group"){
						this.controller.setGroupName(id, txt)
					}
				}
			} else {
				this.logger.log(-1, "onChangeLabel", "No node > ", id);
			}
		},

		onDnd (from, to) {
			this.logger.log(-1, "onDnd", "entry > ", from + ' -> ' + to);

			let fromNode = this.nodes[from]
			let toNode = this.nodes[to]
			if (fromNode && toNode) {
				if (this.controller) {
					this.controller.changeLayer({
						source: fromNode.id,
						widgetID: fromNode.widgetID,
						screenID: fromNode.screenID,
						groupID: fromNode.groupID,
						type: fromNode.type
					}, {
						source: toNode.id,
						widgetID: toNode.widgetID,
						screenID: toNode.screenID,
						groupID: toNode.groupID,
						type: toNode.type
					});
				} else {
					console.warn('No controller')
				}

				if (fromNode.fixed === true && this.canvas) {
					setTimeout (() => {
						this.canvas.showHint('Fixed widgets are always rendered on top.')
					}, 1000)
				}
			}
		},

		render (model){
			this.logger.log(2,"render", "render > ", model);
			this.model = model;
			this.createNestedModel(model);
		},


		createNestedModel (model){
			var result = [];
			this.nodes = {}

			// 1) Build group lookup
			let parentGroups = {}
			for(let id in model.groups){
				let group = model.groups[id]
				for (let i=0; i < group.children.length; i++) {
					let widgetID = group.children[i]
					parentGroups[widgetID] = group
				}
				if (group.groups) {
					for (let i=0; i < group.groups.length; i++){
						let groupID = group.groups[i]
						parentGroups[groupID] = group
					}
				}
			}

			//

			// build a tree for each screen
			for(let id in model.screens){
				let screen = model.screens[id];

				let tree = {
					name: screen.name,
					id: screen.id,
					children: []
				};
				let groupNodes = {};
				let masterNodes = {}
				let sorted = this.getSortedScreenChildren(model, screen)
				for(let i=0; i< sorted.length; i++){
					let widget = sorted[i];

					/**
					 * FIMXE: Make here a extra group for the master widgets
					 */
					if (widget.inherited) {
						let masterScreen = model.screens[widget.masterScreen]
						let master = {
							id: masterScreen.id,
							name: masterScreen.name,
							inherited: true,
							type: 'Master'
						}
						let node = this.createNode(widget, widget.id, screen.id, null, 'widget')
						let masterNode = this.getOrCreateMaster(master, screen.id, masterNodes, tree, widget)
						masterNode.children.push(node)
					} else {

						/**
						 * Check if we have a group
						 */
						if (parentGroups[widget.id]){
							let group = parentGroups[widget.id]
							let node = this.createNode(widget, widget.id, screen.id, group.id, 'widget')
							let groupNode = this.getOrCreateGroup(group, screen.id, groupNodes, parentGroups, tree, widget)
							groupNode.children.push(node)
						} else {
							let node = this.createNode(widget, widget.id, screen.id, null, 'widget')
							tree.children.push(node)
						}
					}
				}
				result.push(tree)
			}

			/**
			 * Now still add before and after listeners
			 */
			this.trees = result
		},

		getSortedScreenChildren (model, screen) {
			let widgets = {}
			for (let i=0; i < screen.children.length; i++){
				let widgetID = screen.children[i];
				let widget = model.widgets[widgetID];
				if (widget) {
					widgets[widget.id] = widget
				}
			}
			return this.getOrderedWidgets(widgets).reverse();
		},

		getOrCreateMaster (masterScreen, screenId, masterNodes, tree, widget) {
			if (!masterNodes[masterScreen.id]) {
				let newMasterNode = this.createNode(masterScreen, widget.id, screenId, null, 'master', false);
				masterNodes[masterScreen.id] = newMasterNode;
				tree.children.push(newMasterNode);
			}
			return masterNodes[masterScreen.id]
		},

		getOrCreateGroup (group, screenId, groupNodes, parentGroups, tree, widget) {
			/**
			 * Check if we have to create a group node, or can recycle one
			 */
			if (!groupNodes[group.id]){


				/**
				 * Check if we have to create parent groups
				 */
				if (parentGroups[group.id]) {
					let parentGroup = parentGroups[group.id]

					let newGroupNode = this.createNode(group, widget.id, screenId, parentGroup.id, 'group');
					groupNodes[group.id] = newGroupNode;

					let parentNode = this.getOrCreateGroup(parentGroup, screenId, groupNodes, parentGroups, tree, widget)
					parentNode.children.push(newGroupNode)
				} else {
					let newGroupNode = this.createNode(group, widget.id, screenId, null, 'group');
					groupNodes[group.id] = newGroupNode;

					tree.children.push(newGroupNode);
				}
			}
			return groupNodes[group.id]
		},

		createNode (box, widgetID, screenID, groupId, type = 'widget', defaultIsOpen = true) {
			if (this.openNodes[box.id] === undefined) {
				this.openNodes[box.id] = defaultIsOpen
			}
			let node = {
				id: box.id,
				widgetID: widgetID,
				screenID: screenID,
				groupID: groupId,
				label: box.name, // + ' (' + box.id + ') ' + box.z,
				icon: this.getNodeIcon(box),
				closeIcon : this.getCloseIcon(box),
				openIcon: this.getOpenIcon(box),
				children:[],
				type: type,
				open: this.openNodes[box.id],
				inherited: box.inherited,
				fixed: false
			}

			if (box.style && box.style.fixed) {
				node.label += ' (fixed)'
				node.fixed = true
			}

			if (box.inherited) {
				node.css = "MatcLayerListWidgetInherited"
				node.disabled = true
			}
			this.nodes[node.id] = node
			this.lastNode = node
			return node;
		},

		getCloseIcon (box) {
			if (box.type == "Master") {
				return "mdi mdi-content-duplicate";
			}
			return false
		},

		getOpenIcon (box) {
			if (box.type == "Master") {
				return "mdi mdi-content-duplicate";
			}
			return false
		},

		getNodeIcon (box){
			if (box.type == "Master") {
				return "mdi mdi-content-duplicate";
			}
			if (box.type == "Label") {
				return "mdi mdi-format-title";
			}
			if (box.type == "Icon") {
				if (box.style.icon){
					return box.style.icon;
				}
			}
			if (box.type == "Table") {
				return "mdi mdi-table-large"
			}
			return "mdi mdi-crop-portrait";
		},

		changeName (box) {
			let node = this.nodes[box.id]
			if (node) {
				this.$set(node, 'label', box.name)
			} else {
				let tree = this.trees.find(t => t.id === box.id)
				if (tree) {
					this.$set(tree, 'name', box.name)
				} else {
					this.logger.error('changeName', 'No node with id', box.id)
				}
			}
		},

		unSelect () {
			this.unSelectNodes()
		},

		selectGroup (groupID){
			this.selectNode([groupID])
		},

		selectWidget (widgetID) {
			this.selectNode([widgetID])
		},

		selectMulti (ids) {
			this.selectNode(ids)
		},

		selectNode (ids) {
			this.unSelectNodes()
			ids.forEach(id => {
				let node = this.nodes[id]
				if (node) {
					this.$set(node, 'selected', true)
					this.$set(node, 'scroll', true)
				}
			})
		},

		unSelectNodes () {
			for (let id in this.nodes) {
				let node = this.nodes[id]
				this.$set(node, 'selected', false)
				this.$set(node, 'scroll', false)
			}
		},

		setSelectedWidget (){
			console.warn('setSelectedWidget() DEPRCATED', new Error().stack)
		},

		setSelectedGroup (){
			console.warn('setSelectedGroup() DEPRCATED', new Error().stack)
		}
	},
	watch: {
		value (v) {
			this.render(v)
		}
	},
    mounted () {
		if (this.value) {
			this.render(this.value)
		}
    }
}
</script>
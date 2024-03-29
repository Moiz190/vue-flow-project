<template>
  <div ref="dropdownRef">
    <custom-dropdown
      :isMenuVisible="isMenuVisible"
      :top="dropdownPosition.top"
      :left="dropdownPosition.left"
    />
  </div>
  <VueFlow
    :nodes="nodes"
    :edges="edges"
    :class="{ dark }"
    class="flow-diagram-container"
    :default-viewport="{ zoom: 1.5 }"
    :min-zoom="0.2"
    :max-zoom="1.2"
  >
    <template #node-custom="customNodeProps">
      <div class="custom-node relative">
        <div
          v-if="Object.keys(customNodeProps.data).length === 0"
          class="h-full flex items-center justify-center add-new-node-container"
        >
          <div
            class="add-new-btn font-bold flex flex-col items-center justify-between"
          >
            <div class="add-new-icon"><span>+</span></div>
            <div><span>Add</span></div>
          </div>
          <div
            class="remove-node-container absolute"
            @click="handleRemoveNode(customNodeProps.id)"
          >
            x
          </div>
          <div
            class="add-right-node add-node-container flex items-center justify-center absolute"
            @click="handleAddNode"
          >
            +
          </div>
          <div
            class="add-left-node add-node-container flex items-center justify-center absolute"
            @click="handleAddNode"
          >
            +
          </div>
        </div>
        <div v-else>
          {{ customNodeProps.id }}
        </div>
      </div>
      <Handle
        v-if="customNodeProps.id !== nodes[nodes.length - 1].id"
        :id="customNodeProps.id"
        type="source"
        :position="Position.Right"
      />
      <Handle
        v-if="customNodeProps.id !== '0'"
        :id="customNodeProps.id"
        type="target"
        :position="Position.Left"
      />
    </template>
    <Background pattern-color="#aaa" :gap="16" />
  </VueFlow>
</template>
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from "vue";
import { Edge, Handle, Node, Position, VueFlow } from "@vue-flow/core";
import CustomDropdown from "./components/CustomDropdown.vue";
import { Background } from "@vue-flow/background";
const isMenuVisible = ref(false);
const dropdownPosition = ref({
  top: 0,
  left: 0,
});
const dropdownRef = ref<null | HTMLElement>(null);

const handleClickOutside = (event: Event) => {
  if (!isClickInsideDropdown(event.target as unknown as Node)) {
    isMenuVisible.value = false;
  }
};
const isClickInsideDropdown = (target: Node) => {
  return dropdownRef.value && dropdownRef.value.contains(target.data);
};
const handleContextMenu = (event: MouseEvent) => {
  isMenuVisible.value = false;
  event.preventDefault();
  dropdownPosition.value.top = event.clientY;
  dropdownPosition.value.left = event.clientX;
  setTimeout(() => {
    isMenuVisible.value = true;
  }, 50);
};

onMounted(() => {
  window.addEventListener("contextmenu", handleContextMenu);
  document.addEventListener("click", handleClickOutside);
});
onUnmounted(() => {
  window.removeEventListener("contextmenu", handleContextMenu);
  document.removeEventListener("click", handleClickOutside);
});

// const isClickInsideDropdown = (target) => {
//   return dropdownRef.value && dropdownRef.value.$el.contains(target);
// };
// const handleClickOutside = (event) => {
//   if (!isClickInsideDropdown(event.target)) {
//     isMenuVisible.value = false;
//   }
// };
// window.addEventListener("contextmenu",
// );
// const handleContextMenu = (event: MouseEvent) => {
//   isMenuVisible.value = false;
//   event.preventDefault();
//   dropdownPosition.value.top = event.clientY;
//   dropdownPosition.value.left = event.clientX;
//   setTimeout(() => {
//     isMenuVisible.value = true;
//   }, 50);
// };
// onMounted(() => {
//   window.addEventListener("contextmenu", handleContextMenu);
//   document.addEventListener("click", handleClickOutside);
// });
const nodes = ref<Node[]>([
  {
    id: "0",
    type: "custom",
    label: "Node 1",
    position: { x: 10, y: 10 },
  },
]);
const edges = ref<Edge[]>([]);
const dark = ref(false);
const nodeCounter = ref(0);
const handleAddNode = () => {
  nodeCounter.value++;
  const newNode = {
    id: nodeCounter.value.toString(),
    type: "custom",
    label: `Node ${nodeCounter.value}`,
    position: {
      x:
        nodes.value.length !== 0
          ? nodes.value[nodes.value.length - 1].position.x + 200
          : 10,
      y:
        nodes.value.length !== 0
          ? nodes.value[nodes.value.length - 1].position.y
          : 10,
    },
    class: "light",
  };
  if (nodes.value.length === 0) {
    nodes.value.push({
      ...newNode,
      sourcePosition: Position.Right,
    });
  } else {
    nodes.value.push({
      ...newNode,
      targetPosition: Position.Left,
      sourcePosition: Position.Right,
    });
    const existingNodeIds = nodes.value.map((node) => node.id);
    const secondSmallerId = secondSmallerNumberThanCounter(
      existingNodeIds,
      nodeCounter.value.toString()
    );
    edges.value.push({
      id: `e${nodeCounter.value}-${secondSmallerId}`,
      source: `${String(secondSmallerId)}`,
      target: `${nodeCounter.value}`,
      type: "smoothstep",
    });
  }
};
const handleRemoveNode = (selectedNodeId: string) => {
  const selectedNodeIndex = nodes.value.findIndex(
    (node) => node.id === selectedNodeId
  );
  if (selectedNodeIndex === -1) return;
  nodes.value.splice(selectedNodeIndex, 1);
  const existingNodesIds = nodes.value.map((node) => node.id);
  if (
    edges.value.length === 1 &&
    (edges.value[0].source === selectedNodeId ||
      edges.value[0].target === selectedNodeId)
  ) {
    edges.value.pop();
    return;
  } else {
    edges.value = edges.value
      .filter((edge) => existingNodesIds.includes(edge.source && edge.target))
      .map((edge) => {
        if (edge.source === selectedNodeId) {
          let newSourceNodeId = findPreviousNode(selectedNodeId);
          return {
            ...edge,
            id: `e${edge.target}-${newSourceNodeId}`,
            source: newSourceNodeId,
          };
        }
        if (edge.target === selectedNodeId) {
          let newTargetNodeId = findNextNode(selectedNodeId);
          return {
            ...edge,
            id: `e${newTargetNodeId}-${edge.source}`,
            target: newTargetNodeId,
          };
        }
        return edge;
      });
    edges.value = removeDuplicatesEdges(
      edges.value.filter((edge) =>
        existingNodesIds.includes(edge.target && edge.source)
      )
    );
  }
};

const removeDuplicatesEdges = (edges: Edge[]) => {
  const uniqueEdges = [];
  const edgeIds = new Set();
  for (const edge of edges) {
    const edgeId = `e${edge.target}-${edge.source}`;
    if (!edgeIds.has(edgeId)) {
      uniqueEdges.push(edge);
      edgeIds.add(edgeId);
    }
  }
  return uniqueEdges;
};
const findNextNode = (nodeId: string): string => {
  let nextNodeId = String(Number(nodeId) + 1);
  let nextNodeIndex = nodes.value.findIndex((node) => node.id === nextNodeId);

  while (nextNodeIndex === -1 && Number(nextNodeId) <= getMaxNodeId()) {
    nextNodeId = String(Number(nextNodeId) + 1);
    nextNodeIndex = nodes.value.findIndex((node) => node.id === nextNodeId);
  }

  return nextNodeId;
};
const findPreviousNode = (nodeId: string): string => {
  let previousNodeId = String(Number(nodeId) - 1);
  let previousNodeIndex = nodes.value.findIndex(
    (node) => node.id === previousNodeId
  );
  while (previousNodeIndex === -1 && Number(previousNodeId) > 0) {
    previousNodeId = String(Number(previousNodeId) - 1);
    previousNodeIndex = nodes.value.findIndex(
      (node) => node.id === previousNodeId
    );
  }
  return previousNodeId;
};
const getMaxNodeId = (): number => {
  return nodes.value.reduce((max, node) => Math.max(max, Number(node.id)), 0);
};
const secondSmallerNumberThanCounter = (arr: string[], counter: string) => {
  const sortedArr = arr.sort(
    (a: string, b: string) => parseInt(a) - parseInt(b)
  );
  const counterIndex = sortedArr.indexOf(counter);
  if (counterIndex === -1) {
    return null;
  }
  return sortedArr[counterIndex - 1];
};

//  <Handle
//   v-if="customNodeProps.id !== nodes[nodes.length - 1].id"
//   :id="customNodeProps.id"
//   type="source"
//   :position="Position.Right"
// />
// <Handle
//   v-if="customNodeProps.id !== '0'"
//   :id="customNodeProps.id"
//   type="target"
//   :position="Position.Left"
// />
// const handleAddNode = () => {
//   nodeCounter.value++;
//   const newNode = {
//     id: nodeCounter.value.toString(),
//     type: "custom",
//     label: `Node ${nodeCounter.value}`,
//     position: {
//       x:
//         nodes.value.length !== 0
//           ? nodes.value[nodes.value.length - 1].position.x + (nodeCounter.value * 200)
//           : 10,
//       y:
//         nodes.value.length !== 0
//           ? nodes.value[nodes.value.length - 1].position.y
//           : 10,
//     },
//     class: "light",
//   };
//   if (nodes.value.length === 0) {
//     addNodes({
//       ...newNode,
//       sourcePosition: Position.Right,
//     });
//   } else {
//     addNodes({
//       ...newNode,
//       targetPosition: Position.Left,
//       sourcePosition: Position.Right,
//     });
//     addEdges({
//       id: `e${nodeCounter.value}-${nodeCounter.value - 1}`,
//       source: `${String(nodeCounter.value - 1)}`,
//       target: `${nodeCounter.value}`,
//       type: "smoothstep",
//     });
//   }
// };

// const handleRemoveNode = (selectedNodeId: string) => {
//   removeNodes(selectedNodeId);
//   removeEdges(selectedNodeId);
// };

// const handleRemoveNode = (selectedNodeId: string) => {
//   const selectedNodeIndex = nodes.value.findIndex(
//     (node) => node.id === selectedNodeId
//   );
//   nodes.value.splice(selectedNodeIndex, 1);
//   if (
//     edges.value.length === 1 &&
//     (edges.value[0].source === selectedNodeId ||
//       edges.value[0].target === selectedNodeId)
//   ) {
//     edges.value.pop();
//     return;
//   } else {
//     edges.value = edges.value.map((edge) => {
//       if (edge.source === selectedNodeId) {
//         const previousNodeId = String(Number(selectedNodeId) - 1);
//         return {
//           ...edge,
//           id: `e${edge.target}-${previousNodeId}`,
//           source: previousNodeId,
//         };
//       }
//       if (edge.target === selectedNodeId) {
//         const nextNodeId = String(Number(selectedNodeId) + 1);
//         return {
//           ...edge,
//           id: `e${nextNodeId}-${edge.source}`,
//           target: nextNodeId,
//         };
//       }
//       return edge;
//     });
//   }
// };
</script>
<style scoped>
.custom-node {
  width: 6.5rem;
  height: 6.5rem;
  background-color: #fff;
  border-radius: 10px;
  box-shadow: 0px 1px 7px 1px rgba(0, 0, 0, 0.4);
}
.add-new-node-container .add-new-icon {
  font-size: 1.3rem;
}
.remove-node-container {
  transition: all 0.5s;
  width: 1.2rem;
  height: 2rem;
  top: 0.7rem;
  right: 1.1rem;
  z-index: -1;
  background-color: #bbbbbb;
  color: #fff;
  padding: 0.3rem;
  border-radius: 0 6px 0;
  box-shadow: 0px 1px 7px 1px rgba(0, 0, 0, 0.4);
}
.add-node-container {
  transition: all 0.5s;
  width: 1.3rem;
  bottom: 0.7rem;
  height: 3.5rem;
  z-index: -1;
  color: #fff;
  box-shadow: 0px 1px 7px 1px rgba(0, 0, 0, 0.4);
  background: linear-gradient(to bottom right, #5721d7, #ba0af3);
  padding: 0.3rem;
}
.add-right-node {
  border-radius: 0 14px 14px 0;
  right: 1.6rem;
}
.add-left-node {
  left: 1.3rem;
  border-radius: 14px 0 0 14px;
}
.add-new-node-container:hover > .add-left-node {
  left: -1.3rem;
}
.add-new-btn:hover,
.remove-node-container:hover,
.add-node-container:hover {
  cursor: pointer;
}
.add-new-node-container:hover > .remove-node-container {
  right: -1.1rem;
}
.add-new-node-container:hover > .add-right-node {
  right: -1.3rem;
}
</style>

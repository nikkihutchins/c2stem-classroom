<template>
  <iframe
    src="https://physdev.c2stem.org"
    id="iframe-id"
    sandbox="allow-scripts allow-same-origin"
    height="100%"
    width="100%"
    frameborder="0"
  ></iframe>
</template>

<script>
function normalizeTree(root) {
  if (root.name === "reportGreaterThan") {
    root.name = "reportLessThan";
    root.next.contained.reverse();
  }

  if (root.name === "reportEquals") {
    root.next.contained.sort((a, b) => {
      return a.name.localeCompare(b.name);
    });
  }

  for (var i = 0; i < root.next.contained.length; i++) {
    if (root.next.contained[i]) normalizeTree(root.next.contained[i]);
  }
  if (root.next.next) normalizeTree(root.next.next);

  return root;
}

const ed = require("edit-distance");
const correctTree = normalizeTree(JSON.parse('{"id":"item_9","name":"receiveGo","modifiedBefore":false,"connectedBefore":false,"next":{"next":{"id":"item_10","name":"doIf","modifiedBefore":false,"connectedBefore":true,"next":{"contained":[{"id":"item_11","name":"reportEquals","modifiedBefore":false,"connectedBefore":true,"next":{"contained":[{"name":"2","modifiedBefore":true,"connectedBefore":false,"next":{"contained":[],"underlay":[]}},{"name":"3","modifiedBefore":true,"connectedBefore":false,"next":{"contained":[],"underlay":[]}}],"underlay":[]}},{"id":"item_16","name":"forward","modifiedBefore":false,"connectedBefore":true,"next":{"contained":[{"name":"10","modifiedBefore":false,"connectedBefore":false,"next":{"contained":[],"underlay":[]}}],"underlay":[]}}],"underlay":[{"name":"","modifiedBefore":false,"connectedBefore":false,"next":{"contained":[],"underlay":[]}}]}},"contained":[],"underlay":[]}}'));

// Map of blocks (id -> block object)
let blocks = {};
// Array of root blocks
let treeRoots = [];
// List of actions for action sequence
let actions = [];

// Tree understanding functions
let insert, remove, update, children;
insert = remove = () => { return 1; };
update = (nodeA, nodeB) => { return (nodeA.name !== nodeB.name ? 1 : 0); };
children = (node) => { 
  let ret = [];
  if (node.next.next) ret.push(node.next.next);
  return [...ret, ...node.next.contained];
};

const actionListener = (action) => {
  // Block class -- used to create standardized objects
  // that represent new blocks in C2STEM
  class Block {
    constructor(id, name, contained) {
      // C2STEM official block id
      // Set to undefined if "ephemeral" block, like a number input (e.g. move *10* steps)
      this.id = id;
      // Block name (from C2STEM)
      // Displayed in graph
      this.name = name;
      // Has this block been modified before? (for epemeral blocks only)
      this.modifiedBefore = false;
      // Has this block been connected before?
      this.connectedBefore = false;
      // Next block (connections)
      this.next = {
        // Next block in linear sequence
        next: undefined,
        // Blocks contained in block -- numbers, sets of statements (like in an if statement), etc.
        contained,
        // Blocks currently hidden in contained.
        // If you put a variable over a number, it gets hidden,
        // and if you remove the variable, the number shows back up.
        // So it needs to be stored, which we do here.
        underlay: [],
      };
    }
  }

  // Create and root a new block
  // Block based on XML in action from C2STEM
  const addBlock = (action) => {
    // Parse XML and get block and id
    const parser = new DOMParser();
    const data = parser.parseFromString(action.args[0], "text/xml");
    console.log(data);
    const blockEl = data.querySelector("block,custom-block");
    const id = blockEl.getAttribute("collabId");

    // Get all blocks "contained" within block
    let contained = [];
    for (let element of data.getElementsByTagName("l")) {
      while (element.children.length > 0) {
        element = element.children[0];
      }

      contained.push(new Block(undefined, element.innerHTML, []));
    }

    // Get name of block
    let name = blockEl.getAttribute("s");
    if (!name) {
      name = "var: " + blockEl.getAttribute("var");
    }

    // Create block, root, and add to block map
    let block = new Block(id, name, contained);
    blocks[id] = block;
    treeRoots.push(block);
    return id;
  };

  // Unroot a block by id
  const unrootBlock = (id) => {
    let i = treeRoots.findIndex((el) => el.id == id);
    if (i != -1) {
      treeRoots.splice(i, 1);
    }
  };

  // Root a child, or a part of a child, based on id
  const rootChildren = (id) => {
    if (id.includes("/")) {
      // If there's a slash in the id,
      // we're rooting blocks that were part of another block
      const t = id.split("/");
      // Root the block
      treeRoots.push(blocks[t[0]].next.contained[t[1]]);
      // If there's an underlay, restore it, otherwise just remove it from the parent
      if (blocks[t[0]].next.underlay[t[1]]) {
        blocks[t[0]].next.contained[t[1]] = blocks[t[0]].next.underlay.splice(t[1], 1)[0];
      } else {
        blocks[t[0]].next.contained.splice(Number(t[1]), 1)[0];
      }
    } else {
      // Root and remove from parent
      treeRoots.push(blocks[id].next.next);
      blocks[id].next.next = undefined;
    }
  };

  let id = action.args[0];
  let actionRep = {
    // Is this a valid action (not a duplicate or unsupported action)?
    valid: true,
    // Action ID
    id: action.id,
    // Action Timestamp
    timestamp: action.time,
    // Action Type (raw, from embedded API)
    rawType: action.type,
    // Action object
    rawAction: action,
    // Type (first, top layer of action)
    type: "construction",
    // Group (second layer of action)
    group: undefined,
    // Type (third layer of action)
    action: undefined,
    // Nearest affected block ID (if relevant)
    block: id,
  };

  // Make decision based on action type
  switch (action.type) {
    // Drag block into window without connecting
    case "addBlock": {
      actionRep.action = "add";
      actionRep.group = "build";
      const id = addBlock(action);
      actionRep.block = id;
      break;
    }
    // Connect block to other block (or drag block into window + immediately connect)
    case "moveBlock": {
      actionRep.action = "connect";

      // If this block is being connected straight from the block pallete,
      // add it to the window first
      if (action.args[0].startsWith("<")) {
        actionRep.group = "build";
        id = addBlock(action);
        blocks[id].connectedBefore = true;
      } else {
        if (!blocks[id].connectedBefore) {
          actionRep.group = "build";
          blocks[id].connectedBefore = true;
        } else {
          actionRep.group = "adjust";
        }
      }

      actionRep.block = id;

      if (typeof action.args[1] == "string") {
        if (typeof action.args[3][1] == "string") {
          rootChildren(action.args[3][1]);
        }

        // If the block is being inserted into a block's condition
        // (e.g. variable into if statement condition), this branch is taken.
        let t = action.args[1].split("/");

        // Get contained block (number after / represents position in contained array)
        // If there's no number, that means the block is replacing the spot of another block
        // (e.g. if we drag a new variable on top of an already inserted variable to replace it),
        // so then we just use the name of the block before the slash, which is the id of the replaced block.
        let containedBlock = t[1]
          ? blocks[t[0]].next.contained[t[1]]
          : blocks[t[0]];

        // If the contained block is an ephemeral block (e.g. just a number) and thus has no id,
        // move it to the underlay, so that way if this new block is ever removed,
        // the number can be restored.
        if (!containedBlock.id) {
          blocks[t[0]].next.underlay[t[1]] = containedBlock;
        } else {
          // Otherwise, root the block that was there before, since it's being displaced
          // by the new block.
          treeRoots.push(containedBlock);
        }

        // If this was a block replacement (described above), then the parent block's position
        // needs to be put in t so we can insert the block into the parent.
        // Otherwise, t already points to the parent.
        if (action.args[4] && action.args[4][1]) {
          t = action.args[4][1].split("/");
        }

        // Insert block into parent's contained and unroot it.
        blocks[t[0]].next.contained[t[1]] = blocks[id];
        unrootBlock(id);
      } else if (action.args[1].loc == "wrap") {
        // If this block is wrapping around other blocks to bring them into their body
        // (e.g. if statement wrapping around pre-existing blocks), this branch is taken.

        if (typeof action.args[4][1] == "object") {
          // If there's a block above the one being inserted,
          // make this its next block.
          blocks[action.args[4][1].element].next.next = blocks[id];
          // Since there's a block above, this isn't a root block, so unroot.
          unrootBlock(id);
        } else {
          // Otherwise, we're wrapping around a root block, so unroot it (this block is now the root).
          unrootBlock(action.args[1].element);
        }

        // Add block we wrapped around to the body of this block.
        blocks[id].next.contained.push(blocks[action.args[1].element]);
      } else {
        // If this block came from the "contained" of another block
        // (e.g. if it used to be in the body of an if statement),
        // root it to get it out of where it came from.
        if (typeof action.args[3][1] == "object") {
          rootChildren(action.args[3][1].element);
        }

        // You can connect multiple blocks at the same time (a stack of blocks).
        // The API will give the first block in the stack, but in order to do
        // an insertion at the bottom, we need the block at the bottom
        // of the stack. So, let's go down the stack until we find the bottom.
        let block = blocks[id];
        while (block.next.next) block = block.next.next;

        // Insertion!
        if (action.args[1].loc == "bottom") {
          // If a block is being inserted under another block,
          // this branch is taken.

          if (action.args[1].element.includes("/")) {
            // If there's a slash, the block is being inserted at the
            // top of the body of another block.
            const t = action.args[1].element.split("/");
            // If there's any more blocks in the body, link them to the end of the stack.
            block.next.next = blocks[t[0]].next.contained[t[1]];
            // Tell the block we're inserting into that this is now the block its body starts with.
            blocks[t[0]].next.contained[t[1]] = blocks[id];
          } else {
            // Get the parent block (the block we're inserting on the bottom of),
            let parent = blocks[action.args[1].element];
            // Take whatever's under it and link it to the bottom of the stack,
            block.next.next = parent.next.next;
            // And link the top of the stack with the block above.
            parent.next.next = blocks[id];
          }

          // Unroot the block, now that it's been inserted.
          unrootBlock(id);
        } else if (action.args[1].loc == "top") {
          // If the block is being inserted on top of another block,
          // where there are no other blocks above it, this branch is taken.

          // Block below this block is our next block
          block.next.next = blocks[action.args[1].element];
          // You can only insert on top of blocks that are roots,
          // so let's unroot the old block.
          unrootBlock(action.args[1].element);
        }
      }

      break;
    }
    // Remove block from canvas
    case "removeBlock": {
      actionRep.action = "remove";
      actionRep.group = "adjust";

      if (typeof action.args[2] == "object") {
        // Normal blocks that have other blocks above them are removed in this branch.

        if (action.args[2].element.includes("/")) {
          // If the block is being removed from the top of a block's body,
          // unlink it from there.
          const t = action.args[2].element.split("/");
          blocks[t[0]].next.contained[t[1]] = blocks[id].next.next;
        } else {
          // Otherwise, unlink it like a normal linked list.
          blocks[action.args[2].element].next.next = blocks[id].next.next;
        }
      } else if (typeof action.args[2] == "string") {
        // Blocks that are part of other block's condition statmements that are being removed
        // take this branch.

        const t = action.args[2].split("/");
        // Replace the block with an empty ephemeral block.
        blocks[t[0]].next.contained[t[1]] = new Block(undefined, "", []);
      } else {
        // Blocks that are on top (root blocks) take this branch.

        // Unroot the block, and move the next block (if it exists) to the root.
        unrootBlock(action.args[0]);
        if (blocks[action.args[0]].next.next) {
          treeRoots.push(blocks[action.args[0]].next.next);
        }
      }

      // Remove all blocks contained in the block (e.g. variable in condition of if statement.)
      blocks[id].next.contained.forEach((block) => delete blocks[block.id]);
      delete blocks[id];
      break;
    }
    // This event is triggerd when a block is disconnected from another block and brought into the canvas,
    // or when the block is simply moved from one place to another on the canvas with no connection changes.
    // We only care about the former case.
    case "setBlockPosition": {
      actionRep.action = "modify";
      actionRep.group = "adjust";

      if (typeof action.args[3][1] == "object") {
        // If the block is a normal block, this branch is taken.

        // Root the child, based on its parent.
        rootChildren(action.args[3][1].element);
      } else if (typeof action.args[3][1] == "string") {
        // If the block is part of a condition statement of another block, this branch is taken.

        // Root the child, based on its parent and its index (in the string, separated by a slash).
        rootChildren(action.args[3][1]);
      } else {
        actionRep.valid = false;
      }

      break;
    }
    // Set numberical field to another number on a block.
    case "setField": {
      actionRep.action = "edit";
      const t = action.args[0].split("/");
      const v = t[2] ? t[2] : t[1];
      let block = blocks[t[0]].next.contained;

      actionRep.block = t[0];

      if (!block[v]) {
        // If field does not currently exist, create it.
        actionRep.group = "build";
        block[v] = new Block(undefined, "", []);
      } else {
        if (!block[v].modifiedBefore) {
          actionRep.group = "build";
          block[v].modifiedBefore = true;
        } else {
          actionRep.group = "adjust";
        }
      }

      block = block[v];
      // Arguments come in either arrays or strings, so assign accordingly.
      if (typeof action.args[1] == "string") {
        if (action.args[1] === action.args[2]) {
          actionRep.valid = false;
        }
        blocks[t[0]].next.contained[v].name = action.args[1];
      } else {
        blocks[t[0]].next.contained[v].name = action.args[1][0];
      }

      break;
    }
    // Toggle boolean field on boolean block.
    case "toggleBoolean": {
      actionRep.action = "edit";
      actionRep.group = "adjust";
      const t = action.args[0].split("/");
      actionRep.block = t[0];
      blocks[t[0]].next.contained[0].name = String(action.args[2]);
      break;
    }
    // ASSESSMENT (USER) ACTIONS
    case "openProject": {
      actionRep.type = "assessment";
      actionRep.group = "analysis";
      actionRep.action = "open";
      break;
    }
    case "pressStart": {
      actionRep.type = "assessment";
      actionRep.group = "visual";
      actionRep.action = "play";
      break;
    }
    case "startScript": {
      actionRep.type = "assessment";
      actionRep.group = "visual";
      actionRep.action = "block";
      break;
    }
    case "addListInput": {
      const t = id.split("/");
      actionRep.group = "build";
      actionRep.block = t[0];
      blocks[t[0]].next.contained.push(new Block(undefined, "", []));
      break;
    }
    case "removeListInput": {
      const t = id.split("/")
      actionRep.group = "adjust";
      actionRep.block = t[0];
      blocks[t[0]].next.contained.pop();
      break;
    }
    default: {
      actionRep.valid = false;
      console.warn(`action not supported: ${action.type}`);
      break;
    }
  }

  // Attach relevant information to action
  actionRep.currentTree = JSON.stringify(treeRoots);
  actionRep.blockMap = JSON.stringify(blocks);

  // Is this a non-assessment action? If so, let's see how close it is to the correct
  // tree, and see if that's better than before (effective/non-effective action).
  // TODO: are we only expecting one tree? how should we choose which one to use for edit distance?
  if (treeRoots[0] && actionRep.type !== "assessment") {
    const ted = ed.ted(normalizeTree(JSON.parse(JSON.stringify(treeRoots[0]))), correctTree, children, insert, remove, update);
    actionRep.ted = ted.distance;
    let lastEditDistance = Infinity;
    for (let i = actions.length - 1; i >= 0; --i) {
      if ("ted" in actions[i]) {
        lastEditDistance = actions[i].ted;
        break;
      }
    }

    actionRep.effective = ted.distance < lastEditDistance;
  }

  if (actionRep.valid) actions.push(actionRep);

  console.log(actions);
  console.log(treeRoots);
  window.localStorage.setItem("blocks", JSON.stringify(blocks));
  window.localStorage.setItem("treeRoots", JSON.stringify(treeRoots));
  window.localStorage.setItem("actionList", JSON.stringify(actions));
};

export default {
  name: "BuildEnv",
  mounted() {
    let ifr_window = document.getElementById("iframe-id");
    this.api = new window.EmbeddedNetsBloxAPI(ifr_window);
    ifr_window.onload = () => {
      this.api.addActionListener(actionListener);
      this.api.addEventListener("startScript", console.log);
    };
  },
  unmounted() {
    //this.api.removeActionListener(actionListener);
  },
};
</script>

<style>
div {
  min-height: 100%;
  height: 100%;
}

iframe {
  width: 100%;
  height: 100%;
  display: block;
}
</style>

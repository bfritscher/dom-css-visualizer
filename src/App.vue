<template>
  <div>
    <h1>DOM|CSS Visualizer</h1>
    <h2>HTML</h2>
    <div>
      <textarea v-model="html" class="html-input"></textarea>
    </div>
    <h2>CSS</h2>
    <div class="input-wrap">
      <textarea divte="off" autocorrect="off" spellcheck="false" class="input" v-model="selector"></textarea>
      <div class="selector">
        <span v-for="part in selectorParts" :class="`type-${part.type}`">{{part.selector}}</span>
      </div>
    </div>
    <h2>Specificity</h2>
    <div class="specificity">
      <span class="type-a">{{specificity.specificityArray[1]}}</span>
      <span class="type-b">{{specificity.specificityArray[2]}}</span>
      <span class="type-c">{{specificity.specificityArray[3]}}</span>
    </div>
    <h2>DOM Tree</h2>
    <div id="tree" ref="tree"></div>
    <div ref="graphOptions"></div>
    <iframe ref="iframe"></iframe>
    <p>
      Inspired by
      <a href="https://specificity.keegan.st/">Specificity Calculator</a>
    </p>
  </div>
</template>

<script>
import { calculate } from "specificity";
import "vis/dist/vis.css";
import vis from "vis";

function buildNetwork(parent, data, parentId = "1") {
  let childIndex = 1;
  for (const tag of [...parent.childNodes].filter(
    n => Node.ELEMENT_NODE === n.nodeType
  )) {
    const id = parentId + childIndex;
    let label = tag.tagName;
    if (tag.id) {
      label += ` #${tag.id}`;
    }
    const classes = [...tag.classList];
    if (classes.length > 0) {
      label += ` .${classes.join(".")}`;
    }
    const node = {
      id,
      label,
      shape: "box",
      group: 0
    };
    data.nodes.push(node);
    data.edges.push({
      from: parentId,
      to: id
    });
    try {
      const style = getComputedStyle(tag);

      if ("50px" === style.getPropertyValue("margin-left")) {
        //span.className = "highlighted";
        node.group = 1;
      }
    } catch (e) {}
    buildNetwork(tag, data, id);
    childIndex++;
  }
  return data;
}

export default {
  name: "HelloWorld",
  props: {
    msg: String
  },
  data() {
    return {
      html: `<h1 class="abc"></h1>
  <ul>
    <li class="test abc"></li>
    <li><img></li>
  </ul>
  <h1 id="abc"></h1>`,
      selector: "li.abc"
    };
  },
  computed: {
    specificity() {
      const res = calculate(this.selector);
      return res && res.length > 0 ? res[0] : { parts: [], specificityArray: [0, 0, 0, 0] };
    },
    selectorParts() {
      let index = 0;
      const parts = [];
      this.specificity.parts.forEach(part => {
        if (part.index > index) {
          parts.push({
            type: "whitespace",
            selector: this.selector.substr(index, part.index - index)
          });
        }
        parts.push(part);
        index = part.index + part.length;
      });
      if (index < this.selector.length) {
        parts.push({
          type: "whitespace",
          selector: `${this.selector.substr(
            index,
            this.selector.length - index
          )} `
        });
      }
      return parts;
    }
  },
  mounted() {
    this.d = this.$refs.iframe.contentDocument;
    this.style = document.createElement("style");
    this.d.head.append(this.style);
    this.build();
  },
  methods: {
    build() {
      this.d.body.innerHTML = this.html;
      this.style.textContent = `${this.selector} { margin-left:50px; }`;
      this.$refs.tree.innerHTML = "";
      //parseTag(this.d.body, this.$refs.tree);
      const data = buildNetwork(this.d.body, {
        nodes: [{ id: "1", label: "body", group: 0 }],
        edges: []
      });
      const options = {
        layout: {
          hierarchical: {
            direction: "UD",
            sortMethod: "directed",
            levelSeparation: 60,
            nodeSpacing: 60,
            parentCentralization: true,
            edgeMinimization: true,
            blockShifting: true
          }
        },
        interaction: { dragNodes: false },
        physics: {
          enabled: false
        },
        nodes: {
          borderWidth: 0,
          font: {
            face: "Lato"
          }
        },
        configure: {
          filter: n => n.includes("direction"),
          container: this.$refs.graphOptions,
          showButton: false
        }
      };
      new vis.Network(
        this.$refs.tree,
        {
          nodes: new vis.DataSet(data.nodes),
          edges: new vis.DataSet(data.edges)
        },
        options
      );
    }
  },
  watch: {
    html() {
      this.build();
    },
    selector() {
      this.build();
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>
@import url("https://fonts.googleapis.com/css?family=Lato:400,700&display=swap");

* {
  box-sizing: border-box;
}

body {
  font-family: "Lato", sans-serif;
}

#tree {
  height: 600px;
}

.highlighted {
  background-color: yellow;
}

iframe {
  display: none;
}
.input-wrap {
  position: relative;
}
.input,
.selector {
  font-size: 28px;
  line-height: 36px;
  font-family: monospace;
  padding: 10px;
}
.input {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
  margin: 0;
  border: none;
  background: none;
  color: #FFF;
  resize: none;
  outline: none;
  overflow: hidden;
}

.html-input {
  width: 100%;
  min-height: 56px;
  color: #fff;
  background: #002B36;
  white-space: pre-wrap;
  word-break: break-word;
}
.input:focus,
.input:active,
.html-input:focus,
.html-input:active {
  box-shadow: 0 0 10px #00171D;
}
.selector {
  min-height: 56px;
  background: #002B36;
  white-space: pre-wrap;
  word-break: break-word;
}
.specificity span {
  height: 42px;
  width: 42px;
  display: inline-block;
  line-height: 42px;
  border-radius: 50%;
  font-size: 28px;
  font-weight: bold;
  color: #FFF;
  text-align: center;
  margin: 2px;
}
.selector .type-a {
  color: #A92B68;
  background: #A92B68;
}
.specificity .type-a {
  background: #A92B68;
}
.selector .type-b {
  color: #1E6FA8;
  background: #1E6FA8;
}
.specificity .type-b {
  background: #1E6FA8;
}
.selector .type-c {
  color: #22817A;
  background: #22817A;
}
.specificity .type-c {
  background: #22817A;
}
</style>

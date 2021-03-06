<template>
  <div id="app" :class="{ embed }">
    <header v-if="!embed">
      <h1>DOM|CSS Visualizer</h1>
    </header>
    <main>
      <section class="col1" v-show="!embed">
        <h2>HTML</h2>
        <div>
          <textarea v-model="html" class="html-input"></textarea>
        </div>
        <h2>CSS</h2>
        <div class="input-wrap">
          <textarea
            divte="off"
            autocorrect="off"
            spellcheck="false"
            class="input"
            v-model="selector"
          ></textarea>
          <div class="selector">
            <span
              v-for="(part, i) in selectorParts"
              :class="`type-${part.type}`"
              :key="i"
              >{{ part.selector }}</span
            >
          </div>
        </div>
        <h2>Specificity</h2>
        <div class="specificity">
          <span class="type-a">{{ specificity.specificityArray[1] }}</span>
          <span class="type-b">{{ specificity.specificityArray[2] }}</span>
          <span class="type-c">{{ specificity.specificityArray[3] }}</span>
        </div>
      </section>
      <section>
        <h2 v-if="!embed">DOM Tree</h2>
        <div id="tree" ref="tree"></div>
        <div ref="graphOptions" v-show="!embed"></div>
      </section>
    </main>
    <footer>
      <iframe ref="iframe"></iframe>
      <p v-if="!embed">
        Inspired by
        <a href="https://specificity.keegan.st/">Specificity Calculator</a>
      </p>
    </footer>
  </div>
</template>

<script>
import { calculate } from "specificity";
import "vis/dist/vis.css";
import vis from "vis";
let visNetwork;

const skipID = "skip_me";

function buildNetwork(parent, data, parentId = "1") {
  let childIndex = 1;
  // eslint-disable-next-line
  for (const tag of [...parent.childNodes].filter(
    (n) => Node.ELEMENT_NODE === n.nodeType && n.id !== skipID
  )) {
    const id = parentId + childIndex;
    let label = `*${tag.tagName.toLowerCase()}*`;
    if (tag.id) {
      label += ` _#${tag.id}_`;
    }
    const classes = [...tag.classList];
    if (classes.length > 0) {
      label += ` _.${classes.join(".")}_`;
    }
    const node = {
      id,
      label,
      shape: "box",
    };
    data.nodes.push(node);
    data.edges.push({
      from: parentId,
      to: id,
    });
    try {
      const style = getComputedStyle(tag);
      // if computed style matches our style definition from build
      if ("50px" === style.getPropertyValue("margin-left")) {
        node.group = "highlighted";
      }
    } catch (e) {
      console.log(e);
    }
    buildNetwork(tag, data, id);
    childIndex++;
  }
  return data;
}

export default {
  name: "App",
  data() {
    return {
      html: `<h1 class="abc"></h1>
<p></p>
<p></p>
<ul>
  <li class="test abc"></li>
  <li><img></li>
</ul>
<p></p>
<h1 id="abc"></h1>`,
      selector: "li.abc",
      embed: false,
    };
  },
  computed: {
    specificity() {
      const res = calculate(this.selector);
      return res && res.length > 0
        ? res[0]
        : { parts: [], specificityArray: [0, 0, 0, 0] };
    },
    selectorParts() {
      let index = 0;
      const parts = [];
      this.specificity.parts.forEach((part) => {
        if (part.index > index) {
          parts.push({
            type: "whitespace",
            selector: this.selector.substr(index, part.index - index),
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
          )} `,
        });
      }
      return parts;
    },
  },
  mounted() {
    this.d = this.$refs.iframe.contentDocument;
    this.style = document.createElement("style");
    this.style.id = skipID;
    this.build();
    //iframe embed support
    if (window.top != window) {
      this.embed = true;
      this.html = "";
      window.addEventListener(
        "message",
        (message) => {
          let txt = message.data;
          if (typeof txt === "object" && "vueDetected" in txt) {
            return;
          }
          if (typeof txt === "object" && "html" in txt) {
            this.html = txt.html;
            return;
          }
        },
        false
      );
    }
    window.addEventListener("resize", () => {
      if (visNetwork) {
        visNetwork.redraw();
      }
    })
  },
  methods: {
    build() {
      if (visNetwork) {
        visNetwork.destroy();
      }
      this.$refs.graphOptions.innerHTML = "";
      let startNode = this.d.body;
      const hasHTML = this.html.toLowerCase().includes("<html");
      if (hasHTML) {
        startNode = this.d.children[0];
      }
      startNode.innerHTML = this.html;
      this.d.head.append(this.style);
      // using this property to detect which element gets selected
      this.style.textContent = `${this.selector} { margin-left:50px; }`;
      this.$refs.tree.innerHTML = "";
      const data = buildNetwork(startNode, {
        nodes: [{ id: "1", label: `*${hasHTML ? "html" : "body"}*` }],
        edges: [],
      });
      const options = {
        autoResize: false,
        height: "100%",
        width: "100%",
        layout: {
          hierarchical: {
            direction: "UD",
            sortMethod: "directed",
            levelSeparation: 100,
            nodeSpacing: 100,
            parentCentralization: true,
            edgeMinimization: true,
            blockShifting: true,
          },
        },
        interaction: { dragNodes: false },
        physics: {
          enabled: false,
        },
        nodes: {
          borderWidth: 0,
          color: {
            background: "#00171d",
          },
          font: {
            color: "white",
            size: 24,
            multi: "md",
            face: "Lato",
          },
        },
        groups: {
          highlighted: {
            font: { color: "black" },
            color: { background: "#FFEB3B" },
          },
        },
        configure: {
          filter: (n) => n.includes("direction"),
          container: this.$refs.graphOptions,
          showButton: false,
        },
      };
      visNetwork = new vis.Network(
        this.$refs.tree,
        {
          nodes: new vis.DataSet(data.nodes),
          edges: new vis.DataSet(data.edges),
        },
        options
      );
    },
  },
  watch: {
    html() {
      this.build();
    },
    selector() {
      this.build();
    },
  },
};
</script>

<style>
@import url("https://fonts.googleapis.com/css?family=Lato:400,700&display=swap");

* {
  box-sizing: border-box;
}

html,
body {
  font-family: "Lato", sans-serif;
  margin: 0;
  height: 100%;
}
#app {
  height: 100%;
  display: flex;
  flex-direction: column;
  padding: 10px;
  position: relative;
}

#app.embed {
  overflow: hidden;
}

header {
  position: absolute;
  right: 20px;
}

h1 {
  margin: 0;
}

h2 {
  margin: 0.5em 0;
}

main {
  flex: 1;
  display: flex;
}
main > section {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 10px;
}

.col1 {
  max-width: 30%;
}

footer {
  font-size: 0.8em;
}

#tree {
  flex: 1;
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
  color: #fff;
  resize: none;
  outline: none;
  overflow: hidden;
}

.html-input {
  width: 100%;
  min-height: 56px;
  color: #fff;
  background: #002b36;
  white-space: pre-wrap;
  word-break: break-word;
}
.input:focus,
.input:active,
.html-input:focus,
.html-input:active {
  box-shadow: 0 0 10px #00171d;
}
.selector {
  min-height: 56px;
  background: #002b36;
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
  color: #fff;
  text-align: center;
  margin: 2px;
}
.selector .type-a {
  color: #a92b68;
  background: #a92b68;
}
.specificity .type-a {
  background: #a92b68;
}
.selector .type-b {
  color: #1e6fa8;
  background: #1e6fa8;
}
.specificity .type-b {
  background: #1e6fa8;
}
.selector .type-c {
  color: #22817a;
  background: #22817a;
}
.specificity .type-c {
  background: #22817a;
}
</style>

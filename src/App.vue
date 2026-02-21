<template>
  <div :class="{ embed }">
    <header v-if="!embed">
      <h1>DOM|CSS Visualizer</h1>
    </header>
    <main>
      <section class="col1" v-show="!embed">
        <h2>HTML</h2>
        <div>
          <textarea
            v-model="html"
            class="html-input"
            aria-label="HTML input"
            placeholder="Enter your HTML here..."
          ></textarea>
        </div>
        <h2>CSS</h2>
        <div class="input-wrap">
          <textarea
            autocomplete="off"
            autocorrect="off"
            spellcheck="false"
            class="input"
            v-model="selector"
            aria-label="CSS selector input"
            placeholder="Enter CSS selector (e.g., div.class, #id, ul > li)"
          ></textarea>
          <div class="selector">
            <span
              v-for="(part, i) in selectorParts"
              :class="`type-${part.type}`"
              :key="i"
              >{{ part.selector }}</span
            >
          </div>
          <div v-if="hasError" class="error-message">
            ⚠️ Invalid CSS selector
          </div>
        </div>
        <h2>Specificity</h2>
        <div class="specificity-container">
          <div
            v-for="(spec, index) in specificity.selectors"
            :key="index"
            class="specificity-row"
          >
            <div class="specificity">
              <span class="type-a">{{ spec.a }}</span>
              <span class="type-b">{{ spec.b }}</span>
              <span class="type-c">{{ spec.c }}</span>
            </div>
            <div class="selector-label" v-if="specificity.selectors.length > 1">
              {{ spec.selector }}
            </div>
          </div>
        </div>
      </section>
      <section>
        <h2 v-if="!embed">
          DOM Tree
          <button 
            @click="toggleLayoutDirection" 
            class="layout-toggle-btn"
            :title="layoutDirection === 'UD' ? 'Switch to Left-Right layout' : 'Switch to Top-Down layout'"
          >
            {{ layoutDirection === 'UD' ? '→' : '↓' }}
          </button>
        </h2>
        <div id="tree" ref="tree"></div>
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
import parser from "postcss-selector-parser";
import { selectorSpecificity } from "@csstools/selector-specificity";
import "vis-network/styles/vis-network.css";
import { Network, DataSet } from "vis-network/standalone";
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
      // Check if element matches the selector by using a custom data attribute
      if (tag.hasAttribute("data-css-matched")) {
        node.group = "highlighted";
      }
    } catch (e) {
      // Silently ignore errors during node matching
    }
    buildNetwork(tag, data, id);
    childIndex++;
  }
  return data;
}

// Helper function to calculate specificity for a selector string
function calculateSpecificity(selectorString) {
  try {
    // For empty selectors
    if (!selectorString || !selectorString.trim()) {
      return { a: 0, b: 0, c: 0, parts: [], selectors: [] };
    }

    // Parse the entire selector (including comma-separated ones)
    const selectorAST = parser().astSync(selectorString);

    // For comma-separated selectors, calculate specificity for each
    let maxSpecificity = { a: 0, b: 0, c: 0 };
    let allSelectors = [];

    selectorAST.each((selector) => {
      const specificity = selectorSpecificity(selector);
      const selectorText = selector.toString().trim();

      allSelectors.push({
        a: specificity.a,
        b: specificity.b,
        c: specificity.c,
        selector: selectorText,
      });

      // Track the highest specificity
      if (
        specificity.a > maxSpecificity.a ||
        (specificity.a === maxSpecificity.a &&
          specificity.b > maxSpecificity.b) ||
        (specificity.a === maxSpecificity.a &&
          specificity.b === maxSpecificity.b &&
          specificity.c > maxSpecificity.c)
      ) {
        maxSpecificity = specificity;
      }
    });

    // Collect parts for visualization
    let allParts = [];

    selectorAST.each((selector) => {
      selector.each((node) => {
        let partType = "c"; // default to element
        let value = node.toString();

        if (node.type === "id") {
          partType = "a";
        } else if (
          node.type === "class" ||
          node.type === "attribute" ||
          node.type === "pseudo"
        ) {
          partType = "b";
        } else if (node.type === "tag") {
          partType = "c";
        } else if (node.type === "combinator" || node.type === "comment") {
          partType = "whitespace";
        } else if (node.type === "universal") {
          partType = "c";
        }

        // Get the proper source index
        const sourceIndex =
          node.sourceIndex !== undefined ? node.sourceIndex : 0;

        allParts.push({
          type: partType,
          selector: value,
          index: sourceIndex,
          length: value.length,
        });
      });
    });

    // Sort parts by index to ensure proper order
    allParts.sort((a, b) => a.index - b.index);

    return {
      a: maxSpecificity.a,
      b: maxSpecificity.b,
      c: maxSpecificity.c,
      parts: allParts,
      selectors: allSelectors,
    };
  } catch (e) {
    console.error("Error parsing selector:", e);
    return { a: 0, b: 0, c: 0, parts: [], selectors: [] };
  }
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
      embed: window.location.search.includes("embed"),
      buildTimeout: null,
      hasError: false,
      layoutDirection: "UD", // 'UD' for Up-Down, 'LR' for Left-Right
    };
  },
  computed: {
    specificity() {
      const result = calculateSpecificity(this.selector);
      this.hasError =
        result.selectors.length === 0 && this.selector.trim() !== "";
      return {
        parts: result.parts,
        specificityArray: [0, result.a, result.b, result.c],
        selectors: result.selectors,
      };
    },
    selectorParts() {
      // Create a map of character positions to types
      const charTypes = new Array(this.selector.length).fill("whitespace");

      this.specificity.parts.forEach((part) => {
        const startIndex = part.index;
        const endIndex = startIndex + part.length;
        for (let i = startIndex; i < endIndex && i < charTypes.length; i++) {
          charTypes[i] = part.type;
        }
      });

      // Build parts by grouping consecutive characters of the same type
      const parts = [];
      let currentType = null;
      let currentStart = 0;

      for (let i = 0; i <= this.selector.length; i++) {
        const charType = i < this.selector.length ? charTypes[i] : null;

        if (charType !== currentType) {
          // Save previous part if it exists
          if (currentType !== null) {
            parts.push({
              type: currentType,
              selector: this.selector.substring(currentStart, i),
            });
          }
          // Start new part
          currentType = charType;
          currentStart = i;
        }
      }

      return parts;
    },
  },
  mounted() {
    this.d = this.$refs.iframe.contentDocument;
    this.style = document.createElement("style");
    this.style.id = skipID;
    
    // Wait for font to load before building (Firefox fix)
    this.$nextTick(() => {
      if (document.fonts && document.fonts.ready) {
        document.fonts.ready.then(() => {
          requestAnimationFrame(() => {
            this.build();
          });
        });
      } else {
        // Fallback if Font Loading API is not supported
        requestAnimationFrame(() => {
          this.build();
        });
      }
    });
    
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
    this.onWindowResize = () => {
      if (!visNetwork) {
        return;
      }
      if (this.embed) {
        this.syncEmbedTreeSize();
        visNetwork.setSize("100%", "100%");
      }
      visNetwork.redraw();
    };
    window.addEventListener("resize", this.onWindowResize);
  },
  methods: {
    syncEmbedTreeSize() {
      if (!this.embed || !this.$refs.tree) {
        return;
      }
      this.$refs.tree.style.height = `${window.innerHeight}px`;
    },
    debouncedBuild() {
      // Debounce the build to avoid excessive re-renders while typing
      clearTimeout(this.buildTimeout);
      this.buildTimeout = setTimeout(() => {
        this.build();
      }, 150);
    },
    toggleLayoutDirection() {
      this.layoutDirection = this.layoutDirection === "UD" ? "LR" : "UD";
      this.build();
    },
    build() {
      if (visNetwork) {
        visNetwork.destroy();
      }
      let startNode = this.d.body;
      const hasHTML = this.html.toLowerCase().includes("<html");
      if (hasHTML) {
        startNode = this.d.children[0];
      }
      startNode.innerHTML = this.html;
      this.d.head.append(this.style);

      // Clear previous markers
      const previousMatches = this.d.querySelectorAll("[data-css-matched]");
      previousMatches.forEach((el) => el.removeAttribute("data-css-matched"));

      // Mark matching elements with data attribute instead of using CSS hack
      try {
        const matches = this.d.querySelectorAll(this.selector);
        matches.forEach((el) => {
          if (el.id !== skipID) {
            el.setAttribute("data-css-matched", "true");
          }
        });
      } catch (e) {
        // Invalid selector, no matches
        console.warn("Invalid selector:", this.selector);
      }

      this.$refs.tree.innerHTML = "";
      const data = buildNetwork(startNode, {
        nodes: [{ id: "1", label: `*${hasHTML ? "html" : "body"}*` }],
        edges: [],
      });
      const options = {
        autoResize: this.embed,
        height: "100%",
        width: "100%",
        layout: {
          hierarchical: {
            direction: this.layoutDirection,
            sortMethod: "directed",
            levelSeparation: 100,
            nodeSpacing: 100,
            parentCentralization: true,
            edgeMinimization: true,
            blockShifting: true,
            shakeTowards: "roots",
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
          margin: {
            top: 10,
            right: 15,
            bottom: 10,
            left: 15,
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
      };
      visNetwork = new Network(
        this.$refs.tree,
        {
          nodes: new DataSet(data.nodes),
          edges: new DataSet(data.edges),
        },
        options
      );

      if (this.embed) {
        requestAnimationFrame(() => {
          this.syncEmbedTreeSize();
          if (visNetwork) {
            visNetwork.setSize("100%", "100%");
            visNetwork.redraw();
          }
        });
      }
    },
  },
  watch: {
    html() {
      this.debouncedBuild();
    },
    selector() {
      this.debouncedBuild();
    },
  },
  beforeUnmount() {
    // Cleanup
    clearTimeout(this.buildTimeout);
    if (this.onWindowResize) {
      window.removeEventListener("resize", this.onWindowResize);
    }
    if (visNetwork) {
      visNetwork.destroy();
    }
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
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}
#app > div {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 10px;
  position: relative;
  min-height: 0;
}

#app > div.embed {
  overflow: hidden;
  position: fixed;
  inset: 0;
  padding: 0;
}

#app > div.embed main {
  height: 100vh;
  min-height: 0;
}

#app > div.embed main > section {
  padding: 0;
  min-height: 0;
}

#app > div.embed footer {
  display: none;
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
  display: flex;
  align-items: center;
  gap: 10px;
}

.layout-toggle-btn {
  background: transparent;
  border: 1px solid rgba(30, 111, 168, 0.3);
  color: rgba(30, 111, 168, 0.6);
  font-size: 16px;
  width: 28px;
  height: 28px;
  border-radius: 4px;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
  padding: 0;
  line-height: 1;
}

.layout-toggle-btn:hover {
  background: rgba(30, 111, 168, 0.1);
  color: #1e6fa8;
  border-color: rgba(30, 111, 168, 0.6);
}

.layout-toggle-btn:active {
  transform: scale(0.95);
}

main {
  flex: 1;
  display: flex;
  overflow: hidden;
}
main > section {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 10px;
}

main > section:first-child {
  overflow-y: auto;
}

main > section:last-child {
  overflow: hidden;
  min-height: 0;
}

.col1 {
  max-width: 30%;
  min-width: 300px;
}

@media (max-width: 768px) {
  main {
    flex-direction: column;
  }
  .col1 {
    max-width: 100%;
  }
  header {
    position: static;
    text-align: center;
  }
}

footer {
  font-size: 0.8em;
}

#tree {
  flex: 1;
}

#app > div.embed #tree {
  height: 100%;
  min-height: 100vh;
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
.error-message {
  color: #ff4444;
  font-size: 14px;
  margin-top: 5px;
  font-weight: bold;
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
.specificity-container {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.specificity-row {
  display: flex;
  align-items: center;
  gap: 10px;
}
.specificity {
  display: flex;
  gap: 4px;
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
}
.selector-label {
  font-family: monospace;
  font-size: 16px;
  color: #666;
  background: #f0f0f0;
  padding: 4px 8px;
  border-radius: 4px;
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

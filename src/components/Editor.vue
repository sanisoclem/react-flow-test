<template>
  <div>
    <div class="wrapper"><div class="node-editor" ref="nodeEditor"></div></div>
    <canvas id="canvasOutput"></canvas>
    <div class="dock"></div>
  </div>
</template>

<script>
import {
  Socket,
  NodeEditor,
  Control,
  Output,
  Input,
  Component,
  Engine,
} from "rete";
import ConnectionPlugin from "rete-connection-plugin";
import VueRenderPlugin from "rete-vue-render-plugin";
import ContextMenuPlugin from "rete-context-menu-plugin";
import AreaPlugin from "rete-area-plugin";
import VueNumControl from "./NumControl.vue";
import ModulePlugin from "rete-module-plugin";
import DockPlugin from "rete-dock-plugin";

export default {
  data() {
    return {
      editor: null,
    };
  },
  async mounted() {
    var numSocket = new Socket("Number value");
    var stringSocket = new Socket("String value");
    var dateSocket = new Socket("Date value");

    class NumControl extends Control {
      constructor(emitter, key, readonly) {
        super(key);
        this.component = VueNumControl;
        this.props = { emitter, ikey: key, readonly };
      }

      setValue(val) {
        this.vueContext.value = val;
      }
    }

    class NumComponent extends Component {
      constructor() {
        super("Number");
      }

      builder(node) {
        var out1 = new Output("num", "Number", numSocket);

        return node
          .addControl(new NumControl(this.editor, "num"))
          .addOutput(out1);
      }

      worker(node, inputs, outputs) {
        outputs["num"] = node.data.num;
      }
    }

    class AddComponent extends Component {
      constructor() {
        super("Add");
      }

      builder(node) {
        var inp1 = new Input("num", "Number", numSocket);
        var inp2 = new Input("num2", "Number2", numSocket);
        var out = new Output("num", "Number", numSocket);

        inp1.addControl(new NumControl(this.editor, "num"));
        inp2.addControl(new NumControl(this.editor, "num2"));

        return node
          .addInput(inp1)
          .addInput(inp2)
          .addControl(new NumControl(this.editor, "preview", true))
          .addOutput(out);
      }

      worker(node, inputs, outputs) {
        var n1 = inputs["num"].length ? inputs["num"][0] : node.data.num1;
        var n2 = inputs["num2"].length ? inputs["num2"][0] : node.data.num2;
        var sum = n1 + n2;

        this.editor.nodes
          .find((n) => n.id == node.id)
          .controls.get("preview")
          .setValue(sum);
        outputs["num"] = sum;
      }
    }

    class InvoiceComponent extends Component {
      constructor() {
        super("Invoice");
      }

      builder(node) {
        var invoiceId = new Input("invoiceId", "InvoiceId", stringSocket);
        var accountId = new Input("accountId", "AccountId", stringSocket);
        var amountDue = new Input("amountDue", "AmountDue", numSocket);
        var dueDate = new Input("dueDate", "DueDate", dateSocket);
        var accountName = new Input("accountName", "AccountName", stringSocket);

        return node
          .addInput(invoiceId)
          .addInput(accountId)
          .addInput(amountDue)
          .addInput(dueDate)
          .addInput(accountName);
      }

      worker() {}
    }

    class FileInputComponent extends Component {
      constructor() {
        super("FileInput");
      }

      builder(node) {
        var invoiceId = new Output("invoiceId", "InvoiceId", stringSocket);
        var accountId = new Output("accountId", "AccountId", stringSocket);
        var amountDue = new Output("amountDue", "AmountDue", numSocket);
        var dueDate = new Output("dueDate", "DueDate", dateSocket);
        var accountName = new Output(
          "accountName",
          "AccountName",
          stringSocket
        );

        return node
          .addOutput(invoiceId)
          .addOutput(accountId)
          .addOutput(amountDue)
          .addOutput(dueDate)
          .addOutput(accountName);
      }

      worker() {}
    }

    var container = this.$refs.nodeEditor;
    var components = [
      new NumComponent(),
      new AddComponent(),
      new InvoiceComponent(),
      new FileInputComponent(),
    ];

    var editor = new NodeEditor("demo@0.1.0", container);
    editor.use(ConnectionPlugin);
    editor.use(VueRenderPlugin);
    editor.use(ContextMenuPlugin);
    editor.use(ModulePlugin);
    editor.use(AreaPlugin);
    editor.use(DockPlugin, {
      container: document.querySelector(".dock"),
      itemClass: "item",
      plugins: [VueRenderPlugin], // render plugins
    });

    var engine = new Engine("demo@0.1.0");

    components.map((c) => {
      editor.register(c);
      engine.register(c);
    });

    var dest = await components[2].createNode();
    var srcFile = await components[3].createNode();

    editor.addNode(dest);
    editor.addNode(srcFile);

    editor.on(
      "process nodecreated noderemoved connectioncreated connectionremoved",
      async () => {
        console.log("process");
        await engine.abort();
        await engine.process(editor.toJSON());
      }
    );

    editor.view.resize();
    AreaPlugin.zoomAt(editor);
    editor.trigger("process");
  },
};
</script>

<style>
.node-editor {
  text-align: left;
  height: calc(100vh - 100px);
  width: 100vw;
}

.node .control input,
.node .input-control input {
  width: 140px;
}

.dock {
    height: 100px;
    overflow-x: auto;
    overflow-y: hidden;
    white-space: nowrap;
}

.item {
    display: inline-block;
    vertical-align: top;
    transform: scale(0.8);
    transform-origin: 50% 0;
}

select,
input {
  width: 100%;
  border-radius: 30px;
  background-color: white;
  padding: 2px 6px;
  border: 1px solid #999;
  font-size: 110%;
  width: 170px;
}
</style>

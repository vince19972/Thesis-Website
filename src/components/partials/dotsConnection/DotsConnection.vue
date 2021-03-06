<template>
  <div :class="['dots-connection', pairingState]" @click="onCanvasClicked">
    <div class="container -flex-column">
      <div class="container__top">
        <text-row
          :connectState="states.connect"
          :pairingState="states.pairingState"
          :startNode="states.startNode"
          :endNode="states.endNode"
          :isHighlighted="states.dynamicTextTarget === 'up'"
          :isDehighlighted="states.dynamicTextTarget === 'down'"
          @onNodeClick="onNodeClicked"
          @onMouseOver="onNodeHovered"
        ></text-row>
      </div>
      <div class="container__center">
        <div class="prompt -full-height -flex-center-between">
          <dynamic-text
            :connectState="states.connect"
            :pairingState="states.pairingState"
            :startNode="states.startNode"
            :endNode="states.endNode"
          ></dynamic-text>
        </div>
      </div>
      <div class="container__btm">
        <text-row
          :connectState="states.connect"
          :pairingState="states.pairingState"
          :startNode="states.startNode"
          :endNode="states.endNode"
          :isHighlighted="states.dynamicTextTarget === 'down'"
          :isDehighlighted="states.dynamicTextTarget === 'up'"
          @onNodeClick="onNodeClicked"
          @onMouseOver="onNodeHovered"
          rowType="roles"
        ></text-row>
      </div>
    </div>
    <div class="connections">
      <dot-line
        :startPoint="lineCoord.start"
        :endPoint="lineCoord.end"
        :pairingState="states.pairingState"
      ></dot-line>
    </div>
    <div class="backgrounds">
      <canvas-background
        :connectState="states.connect"
        :startNode="states.startNode"
        :tempNode="states.hoverNodeTarget"
        :isPaired="states.pairingState"
      ></canvas-background>
    </div>
  </div>
</template>

<script lang="ts">
import { Watch, Component, Vue } from "vue-property-decorator";
import { MouseShape } from "@/store/types/browser";

import TextRow from "./TextRow.vue";
import DotLine from "./DotLine.vue";
import CanvasBackground from "./CanvasBackground.vue";
import DynamicText from "./DynamicText.vue";

import {
  ConnectStates,
  UserEvents,
  NodeTypes,
  Payloads,
  LocalStates,
  PairingStates,
  checkPairingState,
} from "./DotsConnection";

@Component({
  components: {
    TextRow,
    DotLine,
    CanvasBackground,
    DynamicText,
  },
})
export default class DotsConnection extends Vue {
  // computed
  get mouseCoord() {
    return this.$store.getters["browser/mouseCoord"];
  }
  get lineCoord() {
    return {
      start: this.startCoord,
      end: this.endCoord,
    };
  }
  get connectState() {
    return this.states.connect;
  }
  get pairingState() {
    if (this.states.pairingState === PairingStates.Paired) {
      return "-is-paired";
    } else if (this.states.pairingState === PairingStates.NotPaired) {
      return "-not-paired";
    } else {
      return "";
    }
  }

  // data
  states: LocalStates = {
    connect: ConnectStates.Connectionless,
    startNode: null,
    endNode: null,
    pairingState: PairingStates.Pending,
    dynamicTextTarget: null,
    hoverNodeTarget: null,
  };
  startCoord: MouseShape = { x: 0, y: 0 };
  endCoord: MouseShape = { x: 0, y: 0 };

  // state machine
  updateConnectState(toState: ConnectStates) {
    this.states.connect = toState;
  }
  updatePairingState(toState: PairingStates) {
    this.states.pairingState = toState;
  }

  // mutations
  mutateInConnectionless(
    coord: MouseShape,
    userEvents: UserEvents,
    nodeType: NodeTypes | null
  ) {
    this.startCoord = { x: coord.x, y: coord.y };
    this.endCoord = this.startCoord;
    this.updatePairingState(PairingStates.Pending);

    if (userEvents === UserEvents.NodeClicked) {
      this.states.startNode = nodeType;
    }
  }
  mutateInConnecting(
    coord: MouseShape,
    endNodeIsStartNode: boolean,
    userEvents: UserEvents,
    nodeType: NodeTypes | null
  ) {
    switch (userEvents) {
      case UserEvents.NodeClicked:
        if (endNodeIsStartNode) {
          this.endCoord = { x: this.startCoord.x, y: this.startCoord.y };
          this.updateConnectState(ConnectStates.Connectionless);
        } else {
          this.endCoord = { x: coord.x, y: coord.y };
          this.states.endNode = nodeType;
          this.updatePairingState(
            checkPairingState(this.states.startNode, this.states.endNode)
          );
          this.updateConnectState(ConnectStates.Connected);
        }
        break;
      case UserEvents.Connecting:
        this.endCoord = this.mouseCoord;
        this.updatePairingState(PairingStates.Pending);
        break;
    }
  }
  mutateInConnected() {
    if (this.states.pairingState === PairingStates.Paired) {
      this.$store.commit("addConnectedPair", this.states.startNode);
    } else {
      setTimeout(() => {
        this.updateConnectState(ConnectStates.Connectionless);
        this.mutates({});
      }, 6000);
    }
  }
  mutates(payloads: Payloads) {
    // safe check
    const coord = payloads.coord ? payloads.coord : { x: 0, y: 0 };
    const endNodeIsStartNode = payloads.endNodeIsStartNode
      ? payloads.endNodeIsStartNode
      : false;
    const userEvents = payloads.userEvents
      ? payloads.userEvents
      : UserEvents.NodeClicked;
    const nodeType = payloads.nodeType ? payloads.nodeType : null;

    // states
    switch (this.states.connect) {
      case ConnectStates.Connectionless:
        this.mutateInConnectionless(coord, userEvents, nodeType);
        break;
      case ConnectStates.Connecting:
        this.mutateInConnecting(
          coord,
          endNodeIsStartNode,
          userEvents,
          nodeType
        );
        break;
      case ConnectStates.Connected:
        this.mutateInConnected();
        break;
    }
  }

  // methods
  onNodeClicked(nodeType: NodeTypes, coord: { x: number; y: number }) {
    switch (this.states.connect) {
      case ConnectStates.Connectionless:
        this.mutates({ coord, nodeType });
        this.updateConnectState(ConnectStates.Connecting);
        break;
      case ConnectStates.Connecting:
        // eslint-disable-next-line no-case-declarations
        const endNodeIsStartNode = this.states.startNode === nodeType;

        this.mutates({ coord, endNodeIsStartNode, nodeType });
        break;
    }
  }
  onNodeHovered(nodeType: NodeTypes | null) {
    this.states.hoverNodeTarget = nodeType;
  }
  onCanvasClicked(event: any) {
    if (event.target && this.states.pairingState !== PairingStates.Paired) {
      const targetClassNames = event.target.classList;
      const isNode =
        targetClassNames.contains("nodes__node-text") ||
        targetClassNames.contains("nodes__node-dot");

      if (!isNode) {
        this.updateConnectState(ConnectStates.Connectionless);
        this.mutates({});
      }
    }
  }
  onMouseOverDynamicText(target: any) {
    this.states.dynamicTextTarget = target;
  }

  // watcher
  @Watch("mouseCoord")
  getEndCoord() {
    if (this.states.connect !== ConnectStates.Connected) {
      this.mutates({ userEvents: UserEvents.Connecting });
    }
  }
  @Watch("connectState")
  watchConnectState() {
    if (this.states.connect === ConnectStates.Connected) {
      this.mutates({});
    }
  }
}
</script>

<style scoped lang="scss">
@import "@/assets/styles/_fonts.scss";
@import "@/assets/styles/_helpers.scss";

$h-gutter: 2vh;
$container-height: calc(100vh - #{$h-gutter} * 2);
$container-row-height: calc((#{$container-height} - (#{$h-gutter} * 5)) / 6);
$container-height-top: calc(#{$container-row-height} * 4);

.dots-connection {
  height: 100vh;
  width: calc(100vw - 2vw * 2);
  margin: 0 auto;
}

.container {
  position: relative;
  z-index: 1;
  margin: auto;
  padding: $h-gutter 0;
  height: 100%;
}
.container__center {
  flex-grow: 1;
}

.connects {
  height: 100%;
  display: flex;

  &__left,
  &__right {
    flex-grow: 1;
  }
  &__center {
    width: 80%;
  }
}

// connections
.connections {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

// bg canvas
.backgrounds {
  position: absolute;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
}
</style>

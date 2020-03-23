<template>
  <g :transform="rootTransform" :class="{done:isDone, on_hold: isOnHold, in_progress: isInProgress}"
    v-on:dblclick="selectTaskInEditor"
  >
    <rect class="node-background"
          rx="5"
          ry="5"
          :width="width"
          :height="height">
    </rect>
    <g :transform="centerTransform">
      <text text-anchor="middle"
            class="node-label"
            v-for="(line, index) in wrappedText"
            :y="((-wrappedText.length/2) + 0.5 + (index)) + 'em'"
            :key="line"
      >
        {{line}}
      </text>
    </g>
    <!-- Task effort   -->
    <g v-if="effort"
      :transform="`translate(${width/2+10},${height-height/5 + 10})`"
      >
      <rect class="duration-background"
            :width="width/2" :height="height/5"
            rx="5"
            ry="5"
      >
      </rect>
      <g :transform="`translate(${width/4},${height/5/2})`">
        <text text-anchor="middle"
              class="duration-label"
              y="0.4em">
          {{effort}}
        </text>
      </g>
    </g>

    <!-- Finish date -->
    <g v-if="!isDone"
      :transform="`translate(${-10},${height-height/5 + 10})`"
      >
      <rect class="duration-background"
            :width="width/2" :height="height/5"
            rx="5"
            ry="5"
      >
      </rect>
      <g :transform="`translate(${width/4},${height/5/2})`">
        <text text-anchor="middle"
              class="duration-label"
              y="0.4em">
          {{finishDate}}
        </text>
      </g>
    </g>
    <!-- Gravatar icons for all assigned people  -->
    <g
      v-for="(assignment,index) in assignments.slice().reverse()"
      :key="assignment.emailAddress"
      :transform="`translate(${-50 + (assignments.length - index) * 25},-25)`"
    >
      <gravatar
        :email="assignment.emailAddress"
        :size="50"
      >
      </gravatar>
    </g>
  </g>
</template>

<!--suppress SassScssResolvedByNameOnly -->
<style lang="scss">
  rect.node-background, rect.duration-background {
    fill: $blue;
    stroke: $blue-11;
    stroke-width: 2px;
  }

  text.node-label, text.duration-label {
    fill: white;
    font-family: 'Open Sans Condensed', "Helvetica Neue", Helvetica, Arial, sans-serif;
    font-size: 1.5em;
  }

  text.duration-label {
    font-size: 1.0em;
  }

  .in_progress {
    rect.node-background, rect.duration-background {
      fill: $yellow;
      stroke: $yellow-11;
    }
    text.node-label, text.duration-label {
      fill: black;
    }
  }

  .done {
    rect.node-background, rect.duration-background {
      fill: $green;
      stroke: $green-11;
    }
  }

  .on_hold {
    rect.node-background, rect.duration-background {
      fill: $red;
      stroke: $red-11;
    }
  }


</style>

<script lang="ts">
import {Prop, Vue} from 'vue-property-decorator';
import Component from 'vue-class-component';
import {Node} from 'dagre';
import {NodeRadius} from '../Constants';
import {Person, Task, TaskState, FinishDate} from 'project.txt';
import Gravatar from './Gravatar.vue';
import lightFormat from 'date-fns/lightFormat';
import {EventBus} from '../EventBus';

@Component({
  components: {Gravatar}
})
export default class TaskNode extends Vue {
  @Prop({default: null, required: true})
  private node!: Node;

  get wrappedText(): string[] {
    const label = this.task.title as string;
    const words = label.split(/\s+/);

    let lines: string[] = [];
    let line = '';
    while (words.length > 0) {
      const word = words.shift() || '';

      // does the word still fit on the line?
      if (line.length + word.length + 1 < 20) {
        line += ' ' + word;
      } else {
        if (line.length == 0) { // overlong word
          line = word;
          lines.push(line);
          line = '';
        } else {
          lines.push(line);
          line = word;
        }
      }
    }

    if (line.length > 0) {
      lines.push(line);
    }

    return lines;
  }

  get finishDate(): string {
    const finishDate = (this.node.finishDate as FinishDate);
    return lightFormat(finishDate.date, 'yyyy-MM-dd') + (finishDate.hasUnknowns ? '?' : '');
  }

  get task() : Task {
    return this.node.task;
  }

  get isDone() {
    return this.task.state === TaskState.Done;
  }

  get isOnHold() {
    return this.task.state === TaskState.OnHold;
  }

  get isInProgress() {
    return this.task.state === TaskState.InProgress;
  }

  get assignments(): Person[] {
    return this.node.assignments || [];
  }

  /**
   * The transform to be applied to the root element. Positions the node in the drawing.
   */
  get rootTransform() {
    return `translate(${this.left},${this.top})`
  }

  /**
   * The transform to be applied to the centered element. Positions the contents of the node inside of it.
   */
  get centerTransform() {
    return `translate(${this.centerX},${this.centerY})`;
  }

  get width() {
    return NodeRadius;
  }

  get height() {
    return NodeRadius;
  }

  /**
   * The left edge of the node.
   */
  get left() {
    return this.node.x - NodeRadius / 2;
  }

  /**
   * The top edge of the node.
   */
  get top() {
    return this.node.y - NodeRadius / 2;
  }

  /**
   * The node's horizontal center.
   */
  get centerX() {
    return NodeRadius / 2;
  }

  /**
   * The node's vertical center.
   */
  get centerY() {
    return NodeRadius / 2;
  }

  get effort() : string | undefined {
    if (!this.task.effort) {
      return undefined;
    }

    const days = this.task.effort.days;
    const hours = this.task.effort.hours;
    const minutes = this.task.effort.minutes;
    return `${days ? days + 'd' : ''}${hours ? hours + 'h' : ''}${minutes ? minutes + 'm' : ''}`;
  }

  selectTaskInEditor() {
    EventBus.$emit('select-task', this.task.lineNumber );
  }
}
</script>

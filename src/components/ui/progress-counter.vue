<template>
    <div>
        <div class="is-2 is-italic">
            {{ $content("progress", "Progress") }}
        </div>
        <progress
            class="progress is-info is-medium"
            :value="(this.currentPage / this.pageCount) * 100"
            max="100"
        >
            45%
        </progress>
    </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";

@Component({})
export default class ProgressCounter extends Vue {
    @Prop() pageCount: Number;
    @Prop() currentPage: Number;

    data() {
        return {
            count: this.pageCount as Number,
            current: this.currentPage as Number
        };
    }
}
</script>

<template>
    <div>
        <hr/>
        <!-- Step header (only shown if more than one step -->
        <div class="row mb-1" v-if="recipe.steps.length > 1">
            <div class="col col-md-8">
                <h5 class="text-primary">
                    <template v-if="step.name">{{ step.name }}</template>
                    <template v-else>{{ $t("Step") }} {{ index + 1 }}</template>
                    <small style="margin-left: 4px" class="text-muted" v-if="step.time !== 0"><i
                        class="fas fa-user-clock"></i> {{ step_time }}</small>
                    <small v-if="start_time !== ''" class="d-print-none">
                        <b-link :id="`id_reactive_popover_${step.id}`" @click="openPopover" href="#">
                            {{ moment(start_time).add(step.time_offset, "minutes").format("HH:mm") }}
                        </b-link>
                    </small>
                </h5>
            </div>
            <div class="col col-md-4 text-right">
                <b-button
                    @click="details_visible = !details_visible"
                    style="border: none; background: none"
                    class="shadow-none d-print-none"
                    :class="{ 'text-primary': details_visible, 'text-success': !details_visible }"
                >
                    <i class="far fa-check-circle"></i>
                </b-button>
            </div>
        </div>

        <b-collapse id="collapse-1" v-model="details_visible">

            <div class="row">
                <!-- ingredients table -->
                <div class="col col-md-4"
                     v-if="step.ingredients.length > 0 && (recipe.steps.length > 1 || force_ingredients)">
                    <table class="table table-sm">
                        <ingredients-card :steps="[step]" :ingredient_factor="ingredient_factor"
                                          @checked-state-changed="$emit('checked-state-changed', $event)"/>
                    </table>
                </div>

                <div class="col"
                     :class="{ 'col-md-8 col-12': recipe.steps.length > 1, 'col-md-12 col-12': recipe.steps.length <= 1 }">
                    <!-- step text -->
                    <div class="row">
                        <div class="col col-md-12">
                            <compile-component :code="step.ingredients_markdown"
                                               :ingredient_factor="ingredient_factor"></compile-component>
                        </div>
                    </div>

                    <!-- File (preview if image, download else) -->
                    <div class="row" v-if="step.file !== null">
                        <div class="col col-md-12">
                            <template>
                                <div
                                    v-if="step.file.preview !== ''">
                                    <b-img :src="step.file.preview" fluid-grow></b-img>
                                </div>
                                <div v-else>
                                    <a :href="step.file.file_download" target="_blank"
                                       rel="noreferrer nofollow">{{ $t("Download") }}
                                        {{ $t("File") }}</a>
                                </div>
                            </template>
                        </div>
                    </div>
                </div>

            </div>

            <!-- Sub recipe (always full width own row) -->
            <div class="row">
                <div class="col col-md-12">
                    <div class="card" v-if="step.step_recipe_data !== null">
                        <b-collapse id="collapse-1" v-model="details_visible">
                            <div class="card-body">
                                <h2 class="card-title">
                                    <a :href="resolveDjangoUrl('view_recipe', step.step_recipe_data.id)">{{
                                            step.step_recipe_data.name
                                        }}</a>
                                </h2>
                                <div v-for="(sub_step, index) in step.step_recipe_data.steps"
                                     v-bind:key="`substep_${sub_step.id}`">
                                    <step-component
                                        :recipe="step.step_recipe_data"
                                        :step="sub_step"
                                        :ingredient_factor="ingredient_factor"
                                        :index="index"
                                        :start_time="start_time"
                                        :force_ingredients="true"
                                    ></step-component>
                                </div>
                            </div>
                        </b-collapse>
                    </div>
                </div>
            </div>
        </b-collapse>

        <!-- Time popover (not rendered) -->
        <div v-if="start_time !== ''">
            <b-popover :target="`id_reactive_popover_${step.id}`" triggers="click" placement="bottom"
                       :ref="`id_reactive_popover_${step.id}`" :title="$t('Step start time')">
                <div>
                    <b-form-group label="Time" label-for="popover-input-1" label-cols="3" class="mb-1">
                        <b-form-input type="datetime-local" id="popover-input-1" v-model.datetime-local="set_time_input"
                                      size="sm"></b-form-input>
                    </b-form-group>
                </div>
                <div class="row" style="margin-top: 1vh">
                    <div class="col-12" style="text-align: right">
                        <b-button @click="closePopover" size="sm" variant="secondary" style="margin-right: 8px">
                            {{ $t("Cancel") }}
                        </b-button>
                        <b-button @click="updateTime" size="sm" variant="primary">{{ $t("Ok") }}</b-button>
                    </div>
                </div>
            </b-popover>
        </div>
    </div>
</template>

<script>
import {calculateAmount} from "@/utils/utils"

import {GettextMixin} from "@/utils/utils"

import CompileComponent from "@/components/CompileComponent"
import IngredientsCard from "@/components/IngredientsCard"
import Vue from "vue"
import moment from "moment"
import {ResolveUrlMixin, calculateHourMinuteSplit} from "@/utils/utils"

Vue.prototype.moment = moment

export default {
    name: "StepComponent",
    mixins: [GettextMixin, ResolveUrlMixin],
    components: {CompileComponent, IngredientsCard},
    props: {
        step: Object,
        ingredient_factor: Number,
        index: Number,
        recipe: Object,
        start_time: String,
        force_ingredients: {
            type: Boolean,
            default: false,
        },
    },
    computed: {
        step_time: function() {
            return calculateHourMinuteSplit(this.step.time)},
    },
    data() {
        return {
            details_visible: true,
            set_time_input: "",
        }
    },
    mounted() {
        this.set_time_input = moment(this.start_time).add(this.step.time_offset, "minutes").format("yyyy-MM-DDTHH:mm")
    },
    methods: {
        calculateAmount: function (x) {
            // used by the jinja2 template
            return calculateAmount(x, this.ingredient_factor)
        },
        updateTime: function () {
            let new_start_time = moment(this.set_time_input)
                .add(this.step.time_offset * -1, "minutes")
                .format("yyyy-MM-DDTHH:mm")

            this.$emit("update-start-time", new_start_time)
            this.closePopover()
        },
        closePopover: function () {
            this.$refs[`id_reactive_popover_${this.step.id}`].$emit("close")
        },
        openPopover: function () {
            this.$refs[`id_reactive_popover_${this.step.id}`].$emit("open")
        },
    },
}
</script>

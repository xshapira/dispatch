<template>
  <ValidationObserver v-slot="{ invalid, validated }">
    <v-card class="mx-auto ma-4" max-width="600" flat outlined :loading="loading">
      <v-card-text>
        <p class="display-1 text--primary">
          Report Incident
          <v-tooltip bottom>
            <template v-slot:activator="{ on }">
              <v-btn icon v-on="on" @click="copyView"><v-icon>mdi-content-copy</v-icon></v-btn>
            </template>
            <span>Copy current fields as template.</span>
          </v-tooltip>
        </p>
        <p>
          If you suspect an incident and need help, please fill out this form to the best of your
          abilities.
        </p>
        <p v-if="project_faq">
          If you have additional questions, please check out the following FAQ document:
          <a :href="project_faq.weblink" target="_blank" style="text-decoration: none">
            {{ project_faq.name }}
            <v-icon small>open_in_new</v-icon>
          </a>
        </p>
        <v-form>
          <v-container grid-list-md>
            <v-layout wrap>
              <v-flex xs12>
                <ValidationProvider name="Title" rules="required" immediate>
                  <v-textarea
                    v-model="title"
                    slot-scope="{ errors, valid }"
                    :error-messages="errors"
                    :success="valid"
                    label="Title"
                    hint="A brief explanatory title. You can change this later."
                    clearable
                    auto-grow
                    rows="2"
                    required
                  />
                </ValidationProvider>
              </v-flex>
              <v-flex xs12>
                <ValidationProvider name="Description" rules="required" immediate>
                  <v-textarea
                    v-model="description"
                    slot-scope="{ errors, valid }"
                    :error-messages="errors"
                    :success="valid"
                    label="Description"
                    hint="A summary of what you know so far. It's all right if this is incomplete."
                    clearable
                    auto-grow
                    rows="3"
                    required
                  />
                </ValidationProvider>
              </v-flex>
              <v-flex xs12>
                <project-select v-model="project" />
              </v-flex>
              <v-flex xs12>
                <incident-type-select :project="project" v-model="incident_type" />
              </v-flex>
              <v-flex xs12>
                <incident-priority-select :project="project" v-model="incident_priority" />
              </v-flex>
              <v-flex xs12>
                <tag-filter-auto-complete :project="project" v-model="tags" label="Tags" />
              </v-flex>
            </v-layout>
            <template>
              <v-btn
                color="info"
                depressed
                :loading="loading"
                :disabled="invalid || !validated"
                @click="report()"
              >
                Submit
                <template v-slot:loader>
                  <v-progress-linear indeterminate color="white" />
                </template>
              </v-btn>
            </template>
          </v-container>
        </v-form>
      </v-card-text>
    </v-card>
  </ValidationObserver>
</template>

<script>
import { ValidationObserver, ValidationProvider, extend } from "vee-validate"
import { mapActions } from "vuex"
import { mapFields } from "vuex-map-fields"
import { required } from "vee-validate/dist/rules"

import router from "@/router"

import DocumentApi from "@/document/api"
import IncidentPrioritySelect from "@/incident/priority/IncidentPrioritySelect.vue"
import IncidentTypeSelect from "@/incident/type/IncidentTypeSelect.vue"
import ProjectSelect from "@/project/ProjectSelect.vue"
import TagFilterAutoComplete from "@/tag/TagFilterAutoComplete.vue"

extend("required", {
  ...required,
  message: "This field is required",
})

export default {
  name: "ReportSubmissionCard",

  components: {
    ValidationProvider,
    ValidationObserver,
    IncidentTypeSelect,
    IncidentPrioritySelect,
    ProjectSelect,
    TagFilterAutoComplete,
  },

  data() {
    return {
      isSubmitted: false,
      project_faq: null,
    }
  },

  computed: {
    ...mapFields("incident", [
      "selected.incident_priority",
      "selected.incident_type",
      "selected.commander",
      "selected.title",
      "selected.tags",
      "selected.description",
      "selected.conversation",
      "selected.conference",
      "selected.visibility",
      "selected.storage",
      "selected.documents",
      "selected.loading",
      "selected.ticket",
      "selected.project",
      "selected.id",
    ]),
    ...mapFields("route", ["query"]),
  },

  methods: {
    getFAQ() {
      if (this.project) {
        DocumentApi.getAll({
          filter: JSON.stringify({
            and: [
              {
                field: "resource_type",
                op: "==",
                value: "dispatch-faq-reference-document",
              },
              {
                model: "Project",
                field: "name",
                op: "==",
                value: this.project.name,
              },
            ],
          }),
        }).then((response) => {
          if (response.data.items.length) {
            this.project_faq = response.data.items[0]
          }
        })
      }
    },
    copyView: function () {
      let store = this.$store
      this.$copyText(window.location).then(
        function () {
          store.commit(
            "notification_backend/addBeNotification",
            {
              text: "View copied to clipboard.",
            },
            { root: true }
          )
        },
        function () {
          store.commit(
            "notification_backend/addBeNotification",
            {
              text: "Failed to copy view to clipboard.",
              color: "red",
            },
            { root: true }
          )
        }
      )
    },
    ...mapActions("incident", ["report", "get", "resetSelected"]),
  },

  created() {
    if (this.query.project) {
      this.project = { name: this.query.project }
    }

    if (this.query.incident_type) {
      this.incident_type = { name: this.query.incident_type }
    }

    if (this.query.incident_priority) {
      this.incident_priority = { name: this.query.incident_priority }
    }

    this.getFAQ()

    this.$watch(
      (vm) => [vm.project],
      () => {
        this.getFAQ()
      }
    )

    this.$watch(
      (vm) => [vm.project, vm.incident_priority, vm.incident_type],
      () => {
        var queryParams = {
          project: this.project ? this.project.name : null,
          incident_priority: this.incident_priority ? this.incident_priority.name : null,
          incident_type: this.incident_type ? this.incident_type.name : null,
        }
        Object.keys(queryParams).forEach((key) => (queryParams[key] ? {} : delete queryParams[key]))
        router.replace({
          query: queryParams,
        })
      }
    )

    if (this.query.tag) {
      if (Array.isArray(this.query.tag)) {
        this.tags = this.query.tag.map(function (t) {
          return { name: t }
        })
      } else {
        this.tags = [{ name: this.query.tag }]
      }
    }
  },
}
</script>

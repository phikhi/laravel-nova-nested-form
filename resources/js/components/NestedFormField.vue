<template>
  <div class="relative">
    <template v-if="shouldDisplay()">
      <template v-if="field.children.length > 0">
        <card
          :class="{ 'overflow-hidden': field.panel && !index }"
          :key="childIndex"
          v-for="(child, childIndex) in field.children"
        >
          <nested-form-header
            :child="child"
            :field="field"
          />
          <component
            :conditions="conditions"
            :errors="errors"
            :field="childField"
            :index="childIndex"
            :is="getComponentName(childField)"
            :key="childFieldIndex"
            :parent-index="index"
            :resource-id="child.resourceId"
            :resource-name="field.resourceName"
            :via-resource="field.viaResource"
            :via-resource-id="field.viaResourceId"
            @file-deleted="$emit('file-deleted')"
            v-for="(childField, childFieldIndex) in child.fields"
            v-show="child.opened"
          />
        </card>
      </template>

      <div
        class="flex flex-col p-8 items-center justify-center"
        v-else
      >
        <p
          class="text-center my-4 font-bold text-80 text-xl"
        >{{__('No related :pluralLabel yet.', { pluralLabel: field.pluralLabel })}}</p>
        <nested-form-add :field="field" />
      </div>
    </template>

    <div
      class="flex flex-col p-8 items-center justify-center"
      v-else
    >
      <p
        class="text-center my-4 font-bold text-80 text-xl"
      >{{__('You cannot add :pluralLabel.', { pluralLabel: field.pluralLabel })}}</p>
    </div>
  </div>
</template>

<script >
import { FormField, HandlesValidationErrors } from 'laravel-nova'
import NestedFormAdd from './NestedFormAdd'
import NestedFormHeader from './NestedFormHeader'

export default {
  mixins: [FormField, HandlesValidationErrors],

  components: {
    NestedFormAdd,
    NestedFormHeader
  },

  props: {
    resourceName: {
      type: String,
      required: true
    },
    resourceId: {
      type: String | Number,
      required: true
    },
    field: {
      type: Object,
      required: true
    },
    conditions: {
      type: Object,
      default: () => ({})
    },
    index: {
      type: Number,
      required: true
    },
    parentIndex: {
      type: Number
    }
  },
  methods: {
    /**
     * Fill the given FormData object with the field's internal value.
     */
    fill(formData) {
      this.field.children.forEach(child => {
        if (child[this.field.keyName]) {
          formData.append(
            `${child.attribute}[${this.field.keyName}]`,
            child[this.field.keyName]
          )
        }
        child.fields.forEach(field => field.fill(formData))
      })

      const regex = /(.*?(?:\[.*?\])+)(\[.*?)\]((?!\[).+)$/

      for (const [key, value] of formData.entries()) {
        if (key.match(regex)) {
          formData.append(key.replace(regex, '$1$2$3]'), value)
          formData.delete(key)
        }
      }
    },

    /**
     * Update the field's internal value.
     */
    handleChange(value) {
      this.value = value
    },

    /**
     * Whether the current form should be displayed.
     */
    shouldDisplay() {
      if (!this.field.displayIf) {
        return true
      }

      let shouldDisplay = []

      for (let i in this.field.displayIf) {
        const {
          attribute,
          is,
          isNot,
          isNotNull,
          isMoreThan,
          isLessThan,
          isMoreThanOrEqual,
          isLessThanOrEqual,
          includes
        } = this.field.displayIf[i]

        if (attribute) {
          const values = Object.keys(this.conditions)
            .filter(key => key.match(`^${attribute}$`))
            .map(key => this.conditions[key])

          if (typeof is !== 'undefined') {
            shouldDisplay.push(values.every(v => v === is))
          } else if (typeof isNot !== 'undefined') {
            shouldDisplay.push(values.every(v => v !== isNot))
          } else if (isNotNull) {
            shouldDisplay.push(values.every(v => Boolean(v)))
          } else if (isNull) {
            shouldDisplay.push(values.every(v => !Boolean(v)))
          } else if (typeof isMoreThan !== 'undefined') {
            shouldDisplay.push(values.every(v => v > isMoreThan))
          } else if (typeof isLessThan !== 'undefined') {
            shouldDisplay.push(values.every(v => v < isLessThan))
          } else if (typeof isMoreThanOrEqual !== 'undefined') {
            shouldDisplay.push(values.every(v => v >= isMoreThanOrEqual))
          } else if (typeof isLessThanOrEqual !== 'undefined') {
            shouldDisplay.push(values.every(v => v <= isLessThanOrEqual))
          } else if (includes) {
            shouldDisplay.push(values.every(v => v && includes.includes(v)))
          }
        }
      }

      return shouldDisplay.every(should => should)
    },
    /**
     * Get all the fields of the instance.
     */
    setAllAttributeWatchers(instance) {
      if (
        instance.fieldAttribute &&
        typeof this.conditions[instance.fieldAttribute] === 'undefined'
      ) {
        this.field.displayIf
          .filter(field =>
            instance.fieldAttribute.match(`^${field.attribute}$`)
          )
          .forEach(field => {
            const keyToWatch = instance.selectedResourceId
              ? 'selectedResourceId'
              : 'value'

            this.$set(
              this.conditions,
              instance.fieldAttribute,
              instance[keyToWatch]
            )

            instance.$watch(keyToWatch, keyToWatch => {
              this.$set(this.conditions, instance.fieldAttribute, keyToWatch)
            })
          })
      }

      if (instance.$children) {
        instance.$children.map(child => this.setAllAttributeWatchers(child))
      }
    },

    /**
     * Get component name.
     */
    getComponentName(child) {
      return child.prefixComponent ? `form-${child.component}` : child.component
    },

    setConditions() {
      if (this.field.displayIf) {
        this.setAllAttributeWatchers(this.$root)
      }
    }
  },

  watch: {
    'field.children'() {
      this.setConditions()
    }
  },

  mounted() {
    if (this.field.displayIf) {
      this.setConditions()
    }
  }
}
</script>

<template>
  <div>
    <template v-if="leaf">
      <v-select
        v-if="schema.enum"
        v-model="value"
        dense
        :class="leafClasses"
        :items="schema.enum"
        :readonly="schema.readOnly"
        :label="schema.title"
      >
        <template #prepend>
          <v-icon
            v-if="!required"
            color="error"
            @click="$emit('remove')"
          >
            mdi-minus-circle
          </v-icon>
          <v-icon v-else>
            mdi-circle-medium
          </v-icon>
        </template>
      </v-select>
      <component
        :is="stringInputType"
        v-else
        v-model="value"
        :label="schema.title"
        :type="fieldType(schema)"
        :class="leafClasses"
        :readonly="schema.readOnly"
        dense
      >
        <template #prepend>
          <v-icon
            v-if="!required"
            color="error"
            @click="$emit('remove')"
          >
            mdi-minus-circle
          </v-icon>
          <v-icon v-else>
            mdi-circle-medium
          </v-icon>
        </template>
      </component>
    </template>
    <template v-else-if="array">
      <template v-if="schema.items.enum">
        <v-select
          v-model="value"
          dense
          multiple
          :label="schema.title"
          :class="leafClasses"
          :items="schema.items.enum"
        />
      </template>
      <template v-else>
        <meta-node
          v-for="(el, i) in value"
          :key="i"
          v-model="value[i]"
          :schema="schema.items"
          :initial="el"
          :level="level+1"
          array-item
          @remove="removeArrayItem(i)"
        />
        <!-- <v-divider v-if="i !== value.length - 1" :key="i" /> -->
        <v-btn
          color="success"
          dark
          rounded
          class="ml-2"
          icon
          @click="addArrayItem"
        >
          <v-icon left>
            mdi-plus
          </v-icon>
        </v-btn>
      </template>
    </template>
    <template
      v-for="(prop, k) in activeProperties"
      v-else
    >
      <v-card
        :key="k"
        class="ml-4 mb-2 py-2"
        :flat="isLeaf(prop)"
      >
        <v-card-title
          v-if="!isLeaf(prop)"
          class="pt-0"
        >
          <v-icon
            v-if="!(k in requiredProperties)"
            class="mr-2"
            color="error"
            @click="removeObjectItem(k)"
          >
            mdi-minus-circle
          </v-icon>
          {{ prop.title }}
        </v-card-title>
        <meta-node
          v-model="value[k]"
          :schema="prop"
          :initial="value[k]"
          :level="level+1"
          :required="k in requiredProperties"
          @remove="removeObjectItem(k)"
        />
      </v-card>
    </template>
    <v-menu
      v-if="Object.keys(optionalProperties).length"
      offset-y
    >
      <template #activator="{ on }">
        <v-btn
          color="primary"
          dark
          rounded
          class="ml-2"
          v-on="on"
        >
          <v-icon left>
            mdi-plus
          </v-icon>Add Property
        </v-btn>
      </template>
      <v-list>
        <v-list-item
          v-for="key in Object.keys(optionalProperties)"
          :key="key"
          @click="addProperty(key)"
        >
          <v-list-item-title>{{ key }}</v-list-item-title>
        </v-list-item>
      </v-list>
    </v-menu>
  </div>
</template>

<script>
import { VTextField, VTextarea } from 'vuetify/lib';

export default {
  name: 'MetaNode',
  props: {
    arrayItem: {
      type: Boolean,
      required: false,
    },
    required: {
      type: Boolean,
      required: false,
      default: () => false,
    },
    schema: {
      type: Object,
      required: true,
    },
    initial: {
      type: [Object, Array, String, Number],
      default: null,
      required: false,
    },
    level: {
      type: Number,
      required: false,
      default: () => 0,
    },
  },
  data() {
    return {
      value: this.copyValue(this.initial) || this.defaultInitial(),
      addedProperties: {},
      leafClasses: 'ml-2 pr-3 pt-1',
    };
  },
  computed: {
    stringInputType() {
      return this.schema.long ? VTextarea : VTextField;
    },
    leaf() {
      return this.isLeaf(this.schema);
    },
    array() {
      return this.schema.type === 'array';
    },
    requiredProperties() {
      if (this.leaf) {
        return {};
      }

      const props = this.schema.properties || this.schema.items.properties;
      const required = this.schema.required || this.schema.items.required;

      if (!(props && required)) {
        return {};
      }

      return Object.entries(props)
        .filter(([key]) => required.includes(key))
        .reduce((acc, [key, val]) => ({ ...acc, [key]: val }), {});
    },
    optionalProperties() {
      if (this.leaf) {
        return {};
      }

      const props = this.schema.properties || this.schema.items.properties;
      if (!props) {
        return {};
      }

      const requiredKeys = Object.keys(this.requiredProperties);
      const addedKeys = Object.keys(this.addedProperties);
      const optionalKeys = Object.keys(props).filter(
        (x) => !requiredKeys.includes(x) && !addedKeys.includes(x),
      );

      return optionalKeys.reduce(
        (obj, key) => ({ ...obj, [key]: props[key] }),
        {},
      );
    },
    activeProperties() {
      return { ...this.requiredProperties, ...this.addedProperties };
    },
  },
  watch: {
    value: {
      handler() {
        this.$emit('input', this.value);
      },
      deep: true,
    },
  },
  created() {
    if (this.schema.type === 'object') {
      const extra = Object.keys(this.value).filter(
        (key) => !(key in this.requiredProperties) && key in this.optionalProperties,
      );
      extra.forEach((key) => {
        this.addProperty(key);
      });
    }
  },
  methods: {
    copyValue(val) {
      if (val === undefined || val === null) return val;

      if (val instanceof Object && !Array.isArray(val)) {
        return { ...val };
      }
      return val.valueOf();
    },
    emptyItem(type) {
      switch (type) {
        case 'object':
          return {};
        case 'array':
          return [];
        case 'string':
          return '';
        case 'number':
          return 0;
        default:
          return null;
      }
    },
    defaultInitial() {
      return this.emptyItem(this.schema.type);
    },
    isLeaf(obj) {
      return !['object', 'array'].includes(obj.type);
    },
    addArrayItem() {
      this.value.push(this.emptyItem(this.schema.items.type));
    },
    removeArrayItem(index) {
      this.value.splice(index, 1);
    },
    removeObjectItem(key) {
      this.$delete(this.value, key);
      this.$delete(this.addedProperties, key);
    },
    addProperty(key) {
      this.$set(this.addedProperties, key, this.optionalProperties[key]);
    },
    fieldType(schema) {
      if (schema.type === 'number' || schema.type === 'integer') {
        return 'number';
      }
      return schema.type;
    },
  },
};
</script>

<template>
  <DefaultField
    :field="currentField"
    :errors="errors"
    :show-help-text="showHelpText"
    class="simple-repeatable form-field"
  >
    <template #field>
      <div class="flex flex-col" v-bind="extraAttributes">
        <!-- Title columns -->
        <div v-if="rows.length" class="mb-1 o1-w-full o1-flex o1-border-b o1-py-2 dark:o1-border-slate-600">
          <div
            v-for="(rowField, i) in fields"
            :key="i"
            class="o1-font-bold o1-text-90 o1-text-md o1-w-full o1-ml-3 o1-flex"
            :style="{ maxWidth: rowField.nsrWidth || null }"
          >
            {{ rowField.name }}
            <span v-if="rowField.required" class="o1-text-red-500 o1-text-sm o1-pl-1">
              {{ __('*') }}
            </span>

            <nova-translatable-locale-tabs
              v-if="rowField.component === 'translatable-field'"
              class="o1-ml-auto"
              style="padding: 0"
              :locales="rowField.formattedLocales"
              :display-type="rowField.translatable.display_type"
              :active-locale="activeLocales[i] || rowField.formattedLocales[0].key"
              :locales-with-errors="repeatableValidation.locales[rowField.originalAttribute]"
              @tabClick="locale => setAllLocales(`sr-${field.attribute}-${rowField.originalAttribute}`, locale)"
              @doubleClick="locale => setAllLocales(void 0, locale)"
            />
          </div>
        </div>

        <draggable
          v-model="rows"
          :item-key="(el, i) => (el && el[0] && el[0].attribute) || i"
          handle=".vue-draggable-handle"
        >
          <template #item="{ element, index }">
            <div class="simple-repeatable-row o1-relative o1-rounded-md o1-py-2 o1-pl-3">
              <div class="vue-draggable-handle o1-flex o1-justify-center o1-items-center o1-cursor-pointer">
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="22" height="22" class="fill-current">
                  <path d="M4 5h16a1 1 0 0 1 0 2H4a1 1 0 1 1 0-2zm0 6h16a1 1 0 0 1 0 2H4a1 1 0 0 1 0-2zm0 6h16a1 1 0 0 1 0 2H4a1 1 0 0 1 0-2z" />
                </svg>
              </div>

              <!-- ✅ CAMPI IN GRIGLIA -->
              <div class="simple-repeatable-fields-wrapper grid grid-cols-1 sm:grid-cols-2 md:grid-cols-4 gap-4 w-full">
                <component
                  v-for="(rowField, j) in element"
                  :key="j"
                  :is="`form-${rowField.component}`"
                  :field="rowField"
                  :errors="repeatableValidation.errors"
                  :unique-id="getUniqueId(field, rowField)"
                  class="w-full"
                  :style="{ maxWidth: rowField.nsrWidth || null }"
                />
              </div>

              <div
                class="delete-icon o1-flex o1-justify-center o1-items-center o1-cursor-pointer o1-fill-current hover:o1-fill-red-600"
                @click="deleteRow(index)"
                v-if="canDeleteRows"
              >
                <svg xmlns="http://www.w3.org/2000/svg" width="22" height="22" viewBox="0 0 24 24" class="o1-fill-inherit">
                  <path
                    d="M8 6V4c0-1.1.9-2 2-2h4a2 2 0 012 2v2h5a1 1 0 010 2h-1v12a2 2 0 01-2 2H6a2 2 0 01-2-2V8H3a1 1 0 110-2h5zM6 8v12h12V8H6zm8-2V4h-4v2h4zm-4 4a1 1 0 011 1v6a1 1 0 01-2 0v-6a1 1 0 011-1zm4 0a1 1 0 011 1v6a1 1 0 01-2 0v-6a1 1 0 011-1z"
                  />
                </svg>
              </div>
            </div>
          </template>
        </draggable>

        <DefaultButton
          v-if="canAddRows"
          @click="addRow"
          class="add-button btn btn-default btn-primary mt-3"
          :class="{ 'delete-width': canDeleteRows }"
          type="button"
        >
          {{ field.addRowLabel }}
        </DefaultButton>
      </div>
    </template>
  </DefaultField>
</template>

<script>
import Draggable from 'vuedraggable';
import { Errors } from 'form-backend-validation';
import { HandlesValidationErrors, DependentFormField } from 'laravel-nova';
import HandlesRepeatable from '../mixins/HandlesRepeatable';
import _set from 'lodash/set';

export default {
  mixins: [HandlesValidationErrors, HandlesRepeatable, DependentFormField],

  components: { Draggable },

  props: ['resourceName', 'resourceId', 'field'],

  methods: {
    fill(formData) {
      const ARR_REGEX = () => /\[\d+\]$/g;
      const allValues = [];

      for (const row of this.rows) {
        let tempForm = new FormData();
        const rowValues = {};

        row.forEach(field => field.fill(tempForm));

        for (const item of tempForm) {
          let key = item[0];
          let value;

          try {
            value = JSON.parse(item[1]);
          } catch {
            value = item[1];
          }

          if (key.includes('---')) key = key.split('---').slice(1).join('---');
          key = key.replace(/---\d+/, '');

          const isArray = !!key.match(ARR_REGEX());
          if (isArray) {
            const result = ARR_REGEX().exec(key);
            key = `${key.slice(0, result.index)}${key.slice(result.index + result[0].length)}`;
          }

          if (isArray) {
            if (!rowValues[key]) rowValues[key] = [];
            rowValues[key].push(value);
          } else {
            _set(rowValues, key, value);
          }
        }

        allValues.push(rowValues);
      }

      formData.append(this.field.attribute, JSON.stringify(allValues));
    },

    addRow() {
      this.rows.push(this.copyFields(this.field.fields, this.rows.length));
    },

    deleteRow(index) {
      this.rows.splice(index, 1);
    },
  },

  computed: {
    extraAttributes() {
      return { ...this.currentField.extraAttributes };
    },

    repeatableValidation() {
      const fields = this.fields;
      const errors = this.errors.errors;
      const repeaterAttr = this.field.attribute;
      const safeRepeaterAttr = repeaterAttr.replace(/.{16}__/, '');
      const erroredFieldLocales = {};
      const formattedKeyErrors = {};

      for (const field of fields) {
        const fieldAttr = field.originalAttribute;
        const relatedErrors = Object.keys(errors).filter(err =>
          err.match(new RegExp(`^${safeRepeaterAttr}.\\d+.${fieldAttr}`))
        );

        if (field.component === 'translatable-field') {
          erroredFieldLocales[fieldAttr] = relatedErrors
            .map(key => key.split('.').slice(-1))
            .flat();
        }

        relatedErrors.forEach(errorKey => {
          const rowIndex = errorKey.split('.')[1];
          let key = `${repeaterAttr}---${field.originalAttribute}---${rowIndex}`;

          if (field.component === 'translatable-field') {
            key += `.${errorKey.split('.').slice(-1)[0]}`;
          }

          formattedKeyErrors[key] = errors[errorKey];
        });
      }

      return {
        errors: new Errors(formattedKeyErrors),
        locales: erroredFieldLocales,
      };
    },

    canAddRows() {
      return this.currentField.canAddRows && (!this.currentField.maxRows || this.rows.length < this.currentField.maxRows);
    },

    canDeleteRows() {
      return this.currentField.canDeleteRows && (!this.currentField.minRows || this.rows.length > this.currentField.minRows);
    },
  },
};
</script>

<style lang="scss">
.simple-repeatable.form-field {
  .simple-repeatable-row {
    width: 100%;
    margin-left: -46px;

    .delete-icon {
      width: 36px;
      height: 36px;
      margin-right: 10px;
    }

    .vue-draggable-handle {
      width: 36px;
      height: 36px;
      margin-right: 10px;

      &:hover {
        opacity: 0.8;
      }
    }

    &:hover {
      background: var(--40);
    }
  }

  .add-button {
    width: 100%;

    &.delete-width {
      width: calc(100% - 22px);
    }
  }

  .simple-repeatable-fields-wrapper {
    display: grid;
    gap: 1rem;
    width: 100%;
    grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  }
}
</style>

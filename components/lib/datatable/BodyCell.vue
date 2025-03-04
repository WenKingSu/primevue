<template>
    <td v-if="loading" :style="containerStyle" :class="containerClass" role="cell"
        v-bind="{ ...getColumnPTOptions(column, 'root'), ...getColumnPTOptions(column, 'bodyCell') }">
        <component :is="column.children.loading" :data="rowData" :column="column" :field="field" :index="rowIndex"
                   :frozenRow="frozenRow" :loadingOptions="loadingOptions"/>
    </td>
    <td v-else :style="containerStyle" :class="containerClass" @click="onClick" @keydown="onKeyDown" role="cell"
        v-bind="{ ...getColumnPTOptions(column, 'root'), ...getColumnPTOptions(column, 'bodyCell') }">
        <span v-if="responsiveLayout === 'stack'" class="p-column-title"
              v-bind="getColumnPTOptions(column, 'columnTitle')">{{ columnProp('header') }}</span>
        <component v-if="column.children && column.children.body && !d_editing" :is="column.children.body"
                   :data="rowData" :column="column" :field="field" :index="rowIndex" :frozenRow="frozenRow"
                   :editorInitCallback="editorInitCallback"/>
        <component
                v-else-if="column.children && column.children.editor && d_editing"
                :is="column.children.editor"
                :data="editingRowData"
                :column="column"
                :field="field"
                :index="rowIndex"
                :frozenRow="frozenRow"
                :editorSaveCallback="editorSaveCallback"
                :editorCancelCallback="editorCancelCallback"
        />
        <component v-else-if="column.children && column.children.body && !column.children.editor && d_editing"
                   :is="column.children.body" :data="editingRowData" :column="column" :field="field" :index="rowIndex"
                   :frozenRow="frozenRow"/>
        <template v-else-if="columnProp('selectionMode')">
            <DTRadioButton
                    v-if="columnProp('selectionMode') === 'single'"
                    :value="rowData"
                    :name="name"
                    :checked="selected"
                    @change="toggleRowWithRadio($event, rowIndex)"
                    :column="column"
                    :disabled="disabled"
                    :hiddenDisable="hiddenDisabled"
                    :pt="pt"/>
            <DTCheckbox
                    v-else-if="columnProp('selectionMode') === 'multiple'"
                    :value="rowData"
                    :checked="selected"
                    :rowCheckboxIconTemplate="column.children && column.children.rowcheckboxicon"
                    :aria-selected="selected ? true : undefined"
                    @change="toggleRowWithCheckbox($event, rowIndex)"
                    :column="column"
                    :disabled="disabled"
                    :hiddenDisable="hiddenDisabled"
                    :pt="pt"
            />
        </template>
        <template v-else-if="columnProp('rowReorder')">
            <component
                    :is="column.children && column.children.rowreordericon ? column.children.rowreordericon : columnProp('rowReorderIcon') ? 'i' : 'BarsIcon'"
                    :class="['p-datatable-reorderablerow-handle', columnProp('rowReorderIcon')]"/>
        </template>
        <template v-else-if="columnProp('expander')">
            <button v-ripple class="p-row-toggler p-link" type="button" :aria-expanded="isRowExpanded"
                    :aria-controls="ariaControls" :aria-label="expandButtonAriaLabel" @click="toggleRow"
                    v-bind="getColumnPTOptions(column, 'rowToggler')">
                <component v-if="column.children && column.children.rowtogglericon" :is="column.children.rowtogglericon"
                           :rowExpanded="isRowExpanded"/>
                <template v-else>
                    <span v-if="isRowExpanded && expandedRowIcon" :class="['p-row-toggler-icon', expandedRowIcon]"/>
                    <ChevronDownIcon v-else-if="isRowExpanded && !expandedRowIcon" class="p-row-toggler-icon"
                                     v-bind="getColumnPTOptions(column, 'rowTogglerIcon')"/>
                    <span v-else-if="!isRowExpanded && collapsedRowIcon"
                          :class="['p-row-toggler-icon', collapsedRowIcon]"/>
                    <ChevronRightIcon v-else-if="!isRowExpanded && !collapsedRowIcon" class="p-row-toggler-icon"
                                      v-bind="getColumnPTOptions(column, 'rowTogglerIcon')"/>
                </template>
            </button>
        </template>
        <template v-else-if="editMode === 'row' && columnProp('rowEditor')">
            <button v-if="!d_editing" v-ripple class="p-row-editor-init p-link" type="button"
                    :aria-label="initButtonAriaLabel" @click="onRowEditInit"
                    v-bind="getColumnPTOptions(column, 'rowEditorInitButton')">
                <component :is="(column.children && column.children.roweditoriniticon) || 'PencilIcon'"
                           class="p-row-editor-init-icon" v-bind="getColumnPTOptions(column, 'rowEditorInitIcon')"/>
            </button>
            <button v-if="d_editing" v-ripple class="p-row-editor-save p-link" type="button"
                    :aria-label="saveButtonAriaLabel" @click="onRowEditSave"
                    v-bind="getColumnPTOptions(column, 'rowEditorEditButton')">
                <component :is="(column.children && column.children.roweditorsaveicon) || 'CheckIcon'"
                           class="p-row-editor-save-icon" v-bind="getColumnPTOptions(column, 'rowEditorEditIcon')"/>
            </button>
            <button v-if="d_editing" v-ripple class="p-row-editor-cancel p-link" type="button"
                    :aria-label="cancelButtonAriaLabel" @click="onRowEditCancel"
                    v-bind="getColumnPTOptions(column, 'rowEditorCancelButton')">
                <component :is="(column.children && column.children.roweditorcancelicon) || 'TimesIcon'"
                           class="p-row-editor-cancel-icon" v-bind="getColumnPTOptions(column, 'rowEditorCancelIcon')"/>
            </button>
        </template>
        <template v-else>{{ resolveFieldData() }}</template>
    </td>
</template>

<script>
import BaseComponent from 'primevue/basecomponent';
import BarsIcon from 'primevue/icons/bars';
import CheckIcon from 'primevue/icons/check';
import ChevronDownIcon from 'primevue/icons/chevrondown';
import ChevronRightIcon from 'primevue/icons/chevronright';
import PencilIcon from 'primevue/icons/pencil';
import TimesIcon from 'primevue/icons/times';
import OverlayEventBus from 'primevue/overlayeventbus';
import Ripple from 'primevue/ripple';
import {DomHandler, ObjectUtils} from 'primevue/utils';
import RowCheckbox from './RowCheckbox.vue';
import RowRadioButton from './RowRadioButton.vue';

export default {
    name: 'BodyCell',
    extends: BaseComponent,
    emits: ['cell-edit-init', 'cell-edit-complete', 'cell-edit-cancel', 'row-edit-init', 'row-edit-save', 'row-edit-cancel', 'row-toggle', 'radio-change', 'checkbox-change', 'editing-meta-change'],
    props: {
        rowData: {
            type: Object,
            default: null
        },
        column: {
            type: Object,
            default: null
        },
        frozenRow: {
            type: Boolean,
            default: false
        },
        rowIndex: {
            type: Number,
            default: null
        },
        index: {
            type: Number,
            default: null
        },
        isRowExpanded: {
            type: Boolean,
            default: false
        },
        selected: {
            type: Boolean,
            default: false
        },
        disabled: {
            type: Boolean,
            default: false
        },
        hiddenDisabled: {
            type: Boolean,
            default: false
        },
        editing: {
            type: Boolean,
            default: false
        },
        editingMeta: {
            type: Object,
            default: null
        },
        editMode: {
            type: String,
            default: null
        },
        responsiveLayout: {
            type: String,
            default: 'stack'
        },
        virtualScrollerContentProps: {
            type: Object,
            default: null
        },
        ariaControls: {
            type: String,
            default: null
        },
        name: {
            type: String,
            default: null
        },
        expandedRowIcon: {
            type: String,
            default: null
        },
        collapsedRowIcon: {
            type: String,
            default: null
        }
    },
    documentEditListener: null,
    selfClick: false,
    overlayEventListener: null,
    data() {
        return {
            d_editing: this.editing,
            styleObject: {}
        };
    },
    watch: {
        editing(newValue) {
            this.d_editing = newValue;
        },
        '$data.d_editing': function (newValue) {
            this.$emit('editing-meta-change', {
                data: this.rowData,
                field: this.field || `field_${this.index}`,
                index: this.rowIndex,
                editing: newValue
            });
        }
    },
    mounted() {
        if (this.columnProp('frozen')) {
            this.updateStickyPosition();
        }
    },
    updated() {
        if (this.columnProp('frozen')) {
            this.updateStickyPosition();
        }

        if (this.d_editing && (this.editMode === 'cell' || (this.editMode === 'row' && this.columnProp('rowEditor')))) {
            setTimeout(() => {
                const focusableEl = DomHandler.getFirstFocusableElement(this.$el);

                focusableEl && focusableEl.focus();
            }, 1);
        }
    },
    beforeUnmount() {
        if (this.overlayEventListener) {
            OverlayEventBus.off('overlay-click', this.overlayEventListener);
            this.overlayEventListener = null;
        }
    },
    methods: {
        columnProp(prop) {
            return ObjectUtils.getVNodeProp(this.column, prop);
        },
        getColumnPTOptions(column, key) {
            return this.ptmo(this.getColumnProp(column), key, {
                props: column.props,
                parent: {
                    props: this.$props,
                    state: this.$data
                }
            });
        },
        getColumnProp(column) {
            return column.props && column.props.pt ? column.props.pt : undefined; //@todo
        },
        resolveFieldData() {
            return ObjectUtils.resolveFieldData(this.rowData, this.field);
        },
        toggleRow(event) {
            this.$emit('row-toggle', {
                originalEvent: event,
                data: this.rowData
            });
        },
        toggleRowWithRadio(event, index) {
            this.$emit('radio-change', {originalEvent: event.originalEvent, index: index, data: event.data});
        },
        toggleRowWithCheckbox(event, index) {
            this.$emit('checkbox-change', {originalEvent: event.originalEvent, index: index, data: event.data});
        },
        isEditable() {
            return this.column.children && this.column.children.editor != null;
        },
        bindDocumentEditListener() {
            if (!this.documentEditListener) {
                this.documentEditListener = (event) => {
                    if (!this.selfClick) {
                        this.completeEdit(event, 'outside');
                    }

                    this.selfClick = false;
                };

                document.addEventListener('click', this.documentEditListener);
            }
        },
        unbindDocumentEditListener() {
            if (this.documentEditListener) {
                document.removeEventListener('click', this.documentEditListener);
                this.documentEditListener = null;
                this.selfClick = false;
            }
        },
        switchCellToViewMode() {
            this.d_editing = false;
            this.unbindDocumentEditListener();
            OverlayEventBus.off('overlay-click', this.overlayEventListener);
            this.overlayEventListener = null;
        },
        onClick(event) {
            if (this.editMode === 'cell' && this.isEditable()) {
                this.selfClick = true;

                if (!this.d_editing) {
                    this.d_editing = true;
                    this.bindDocumentEditListener();
                    this.$emit('cell-edit-init', {
                        originalEvent: event,
                        data: this.rowData,
                        field: this.field,
                        index: this.rowIndex
                    });

                    this.overlayEventListener = (e) => {
                        if (this.$el && this.$el.contains(e.target)) {
                            this.selfClick = true;
                        }
                    };

                    OverlayEventBus.on('overlay-click', this.overlayEventListener);
                }
            }
        },
        completeEdit(event, type) {
            const completeEvent = {
                originalEvent: event,
                data: this.rowData,
                newData: this.editingRowData,
                value: this.rowData[this.field],
                newValue: this.editingRowData[this.field],
                field: this.field,
                index: this.rowIndex,
                type: type,
                defaultPrevented: false,
                preventDefault: function () {
                    this.defaultPrevented = true;
                }
            };

            this.$emit('cell-edit-complete', completeEvent);

            if (!completeEvent.defaultPrevented) {
                this.switchCellToViewMode();
            }
        },
        onKeyDown(event) {
            if (this.editMode === 'cell') {
                switch (event.code) {
                    case 'Enter':
                        this.completeEdit(event, 'enter');
                        break;

                    case 'Escape':
                        this.switchCellToViewMode();
                        this.$emit('cell-edit-cancel', {
                            originalEvent: event,
                            data: this.rowData,
                            field: this.field,
                            index: this.rowIndex
                        });
                        break;

                    case 'Tab':
                        this.completeEdit(event, 'tab');

                        if (event.shiftKey) this.moveToPreviousCell(event);
                        else this.moveToNextCell(event);
                        break;

                    default:
                        break;
                }
            }
        },
        moveToPreviousCell(event) {
            let currentCell = this.findCell(event.target);
            let targetCell = this.findPreviousEditableColumn(currentCell);

            if (targetCell) {
                DomHandler.invokeElementMethod(targetCell, 'click');
                event.preventDefault();
            }
        },
        moveToNextCell(event) {
            let currentCell = this.findCell(event.target);
            let targetCell = this.findNextEditableColumn(currentCell);

            if (targetCell) {
                DomHandler.invokeElementMethod(targetCell, 'click');
                event.preventDefault();
            }
        },
        findCell(element) {
            if (element) {
                let cell = element;

                while (cell && !DomHandler.hasClass(cell, 'p-cell-editing')) {
                    cell = cell.parentElement;
                }

                return cell;
            } else {
                return null;
            }
        },
        findPreviousEditableColumn(cell) {
            let prevCell = cell.previousElementSibling;

            if (!prevCell) {
                let previousRow = cell.parentElement.previousElementSibling;

                if (previousRow) {
                    prevCell = previousRow.lastElementChild;
                }
            }

            if (prevCell) {
                if (DomHandler.hasClass(prevCell, 'p-editable-column')) return prevCell;
                else return this.findPreviousEditableColumn(prevCell);
            } else {
                return null;
            }
        },
        findNextEditableColumn(cell) {
            let nextCell = cell.nextElementSibling;

            if (!nextCell) {
                let nextRow = cell.parentElement.nextElementSibling;

                if (nextRow) {
                    nextCell = nextRow.firstElementChild;
                }
            }

            if (nextCell) {
                if (DomHandler.hasClass(nextCell, 'p-editable-column')) return nextCell;
                else return this.findNextEditableColumn(nextCell);
            } else {
                return null;
            }
        },
        isEditingCellValid() {
            return DomHandler.find(this.$el, '.p-invalid').length === 0;
        },
        onRowEditInit(event) {
            this.$emit('row-edit-init', {
                originalEvent: event,
                data: this.rowData,
                newData: this.editingRowData,
                field: this.field,
                index: this.rowIndex
            });
        },
        onRowEditSave(event) {
            this.$emit('row-edit-save', {
                originalEvent: event,
                data: this.rowData,
                newData: this.editingRowData,
                field: this.field,
                index: this.rowIndex
            });
        },
        onRowEditCancel(event) {
            this.$emit('row-edit-cancel', {
                originalEvent: event,
                data: this.rowData,
                newData: this.editingRowData,
                field: this.field,
                index: this.rowIndex
            });
        },
        editorInitCallback(event) {
            this.$emit('row-edit-init', {
                originalEvent: event,
                data: this.rowData,
                newData: this.editingRowData,
                field: this.field,
                index: this.rowIndex
            });
        },
        editorSaveCallback(event) {
            if (this.editMode === 'row') {
                this.$emit('row-edit-save', {
                    originalEvent: event,
                    data: this.rowData,
                    newData: this.editingRowData,
                    field: this.field,
                    index: this.rowIndex
                });
            } else {
                this.completeEdit(event, 'enter');
            }
        },
        editorCancelCallback(event) {
            if (this.editMode === 'row') {
                this.$emit('row-edit-cancel', {
                    originalEvent: event,
                    data: this.rowData,
                    newData: this.editingRowData,
                    field: this.field,
                    index: this.rowIndex
                });
            } else {
                this.switchCellToViewMode();
                this.$emit('cell-edit-cancel', {
                    originalEvent: event,
                    data: this.rowData,
                    field: this.field,
                    index: this.rowIndex
                });
            }
        },
        updateStickyPosition() {
            if (this.columnProp('frozen')) {
                let align = this.columnProp('alignFrozen');

                if (align === 'right') {
                    let right = 0;
                    let next = this.$el.nextElementSibling;

                    if (next) {
                        right = DomHandler.getOuterWidth(next) + parseFloat(next.style.right || 0);
                    }

                    this.styleObject.right = right + 'px';
                } else {
                    let left = 0;
                    let prev = this.$el.previousElementSibling;

                    if (prev) {
                        left = DomHandler.getOuterWidth(prev) + parseFloat(prev.style.left || 0);
                    }

                    this.styleObject.left = left + 'px';
                }
            }
        },
        getVirtualScrollerProp(option) {
            return this.virtualScrollerContentProps ? this.virtualScrollerContentProps[option] : null;
        }
    },
    computed: {
        editingRowData() {
            return this.editingMeta[this.rowIndex] ? this.editingMeta[this.rowIndex].data : this.rowData;
        },
        field() {
            return this.columnProp('field');
        },
        containerClass() {
            return [
                this.columnProp('bodyClass'),
                this.columnProp('class'),
                {
                    'p-selection-column': this.columnProp('selectionMode') != null,
                    'p-editable-column': this.isEditable(),
                    'p-cell-editing': this.d_editing,
                    'p-frozen-column': this.columnProp('frozen')
                }
            ];
        },
        containerStyle() {
            let bodyStyle = this.columnProp('bodyStyle');
            let columnStyle = this.columnProp('style');

            return this.columnProp('frozen') ? [columnStyle, bodyStyle, this.styleObject] : [columnStyle, bodyStyle];
        },
        loading() {
            return this.getVirtualScrollerProp('loading');
        },
        loadingOptions() {
            const getLoaderOptions = this.getVirtualScrollerProp('getLoaderOptions');

            return (
                getLoaderOptions &&
                getLoaderOptions(this.rowIndex, {
                    cellIndex: this.index,
                    cellFirst: this.index === 0,
                    cellLast: this.index === this.getVirtualScrollerProp('columns').length - 1,
                    cellEven: this.index % 2 === 0,
                    cellOdd: this.index % 2 !== 0,
                    column: this.column,
                    field: this.field
                })
            );
        },
        expandButtonAriaLabel() {
            return this.$primevue.config.locale.aria ? (this.isRowExpanded ? this.$primevue.config.locale.aria.expandRow : this.$primevue.config.locale.aria.collapseRow) : undefined;
        },
        initButtonAriaLabel() {
            return this.$primevue.config.locale.aria ? this.$primevue.config.locale.aria.editRow : undefined;
        },
        saveButtonAriaLabel() {
            return this.$primevue.config.locale.aria ? this.$primevue.config.locale.aria.saveEdit : undefined;
        },
        cancelButtonAriaLabel() {
            return this.$primevue.config.locale.aria ? this.$primevue.config.locale.aria.cancelEdit : undefined;
        }
    },
    components: {
        DTRadioButton: RowRadioButton,
        DTCheckbox: RowCheckbox,
        ChevronDownIcon: ChevronDownIcon,
        ChevronRightIcon: ChevronRightIcon,
        BarsIcon: BarsIcon,
        PencilIcon: PencilIcon,
        CheckIcon: CheckIcon,
        TimesIcon: TimesIcon
    },
    directives: {
        ripple: Ripple
    }
};
</script>

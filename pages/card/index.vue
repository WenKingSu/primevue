<template>
    <DocComponent title="Vue Card Component" header="Card" description="Card is a flexible container component." :componentDocs="docs" :apiDocs="['Card']" :ptTabComponent="ptComponent" />
</template>

<script>
import AccessibilityDoc from '@/doc/card/AccessibilityDoc';
import AdvancedDoc from '@/doc/card/AdvancedDoc';
import BasicDoc from '@/doc/card/BasicDoc';
import ImportDoc from '@/doc/card/ImportDoc';
import StyleDoc from '@/doc/card/StyleDoc';
import PTComponent from '@/doc/card/pt/index.vue';

export default {
    data() {
        return {
            docs: [
                {
                    id: 'import',
                    label: 'Import',
                    component: ImportDoc
                },
                {
                    id: 'basic',
                    label: 'Basic',
                    component: BasicDoc
                },
                {
                    id: 'advanced',
                    label: 'Advanced',
                    component: AdvancedDoc
                },
                {
                    id: 'style',
                    label: 'Style',
                    component: StyleDoc
                },
                {
                    id: 'accessibility',
                    label: 'Accessibility',
                    component: AccessibilityDoc
                }
            ],
            ptComponent: PTComponent
        };
    }
};
</script>

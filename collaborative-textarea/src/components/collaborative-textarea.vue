<template>
    <div>
        <br />
        <!-- There are a lot of different ways content can be inserted into a textarea
            and not all of them trigger a onBeforeInput event. To ensure we are grabbing
            the correct selection before we modify the shared string we need to make sure
            updateSelection is being called for multiple cases. -->
        <textarea ref="textArea"
            rows="20"
            cols="50"
            @beforeinput="updateSelection()"
            @keydown="updateSelection()"
            @click="updateSelection()"
            @contextmenu="updateSelection()"
            @input="handleChange($event)"
            :value="text"></textarea>
    </div>
</template>

<script lang="ts">
import { defineComponent, ref, Ref, toRefs, onMounted, onUnmounted, watch } from 'vue';
import { SharedString } from '@fluidframework/sequence';
import { getObjectType } from '../shared/propTypes';
import { FluidLoaderService } from '../services/fluidLoaderService';
import { CollaborativeText } from '../services/collaborative-text.dataobject';
import { CollaborativeTextContainerRuntimeFactory } from '../services/containerCode';

export default defineComponent({
  name: 'CollaborativeTextArea',
  props: {
      sharedString: SharedString
  },
  setup(props) {
    const textArea: Ref<HTMLTextAreaElement | undefined> = ref();
    const text: Ref<string | undefined> = ref('');
    let sharedString: SharedString | undefined;
    let selectionEnd = 0;
    let selectionStart = 0;

    onMounted(async() => {
      const fluidService = new FluidLoaderService();
      const dataObject = await fluidService.loadDataObject<CollaborativeText>(CollaborativeTextContainerRuntimeFactory);
      sharedString = dataObject.text;
      if (sharedString) {
        text.value = sharedString.getText();
        // Sets an event listener so we can update our state as the value changes
        sharedString.on('sequenceDelta', sequenceDeltaChanged);
      }
    });

    function sequenceDeltaChanged(event: any) {
        const newText = (sharedString) ? sharedString.getText() : '';
        // We only need to insert if the text changed.
        if (newText === text.value) {
            return;
        }

        // If the event is our own then just insert the text
        if (event.isLocal) {
            text.value = newText;
            return;
        }

        // Because we did not make the change we need to manage the remote
        // character insertion.
        const remoteCaretStart = event.first.position;
        const remoteCaretEnd = event.last.position + event.last.segment.cachedLength;
        const textLength = (text && text.value) ? text.value.length : 0;
        const charactersModifiedCount = newText.length - textLength;

        updateSelection();
        const currentCaretStart = selectionStart;
        const currentCaretEnd = selectionEnd;

        let newCaretStart = 0;
        let newCaretEnd = 0;

        // Remote text inserted/removed after our cp range
        if (currentCaretEnd <= remoteCaretStart) {
            // cp stays where it was before.
            newCaretStart = currentCaretStart;
            newCaretEnd = currentCaretEnd;
        } else if (currentCaretStart > (remoteCaretEnd - 1)) {
            // Remote text inserted/removed before our cp range
            // We need to move our cp the number of characters inserted/removed
            // to ensure we are in the same position
            newCaretStart = currentCaretStart + charactersModifiedCount;
            newCaretEnd = currentCaretEnd + charactersModifiedCount;
        } else {
            // Remote text is overlapping cp

            // The remote changes occurred inside current selection
            if (remoteCaretEnd <= currentCaretEnd && remoteCaretStart > currentCaretStart) {
                // Our selection needs to include remote changes
                newCaretStart = currentCaretStart;
                newCaretEnd = currentCaretEnd + charactersModifiedCount;
            } else if (remoteCaretEnd >= currentCaretEnd && remoteCaretStart <= currentCaretStart) {
                // The remote changes encompass our location

                // Our selection has been removed
                // Move our cp to the beginning of the new text insertion
                newCaretStart = remoteCaretStart;
                newCaretEnd = remoteCaretStart;
            } else {
                // We have partial overlapping selection with the changes.
                // This makes things a lot harder to manage so for now we will just remove the current selection
                // and place it to the remote caret start.
                newCaretStart = remoteCaretStart;
                newCaretEnd = remoteCaretStart;
            }
        }

        text.value = newText;
        setCaretPosition(newCaretStart, newCaretEnd);
    }

    function setCaretPosition(newStart: number, newEnd: number) {
        if (textArea && textArea.value) {
            textArea.value.selectionStart = newStart;
            textArea.value.selectionEnd = newEnd;
        }
    }

    function updateSelection() {
        if (textArea && textArea.value) {
            selectionStart = textArea.value.selectionStart ? textArea.value.selectionStart : 0;
            selectionEnd = textArea.value.selectionEnd ? textArea.value.selectionEnd : 0;
        }
    }

    function handleChange(event: any) {
        // We need to set the value here to keep the input responsive to the user
        const currentTarget = event.currentTarget;
        const newText = currentTarget.value;
        const textLength = (text && text.value) ? text.value.length : 0;
        const charactersModifiedCount = textLength - newText.length;
        text.value = newText;

        // Get the new caret position and use that to get the text that was inserted
        const newPosition = currentTarget.selectionStart ? currentTarget.selectionStart : 0;
        const isTextInserted = newPosition - selectionStart > 0;
        if (sharedString) {
            if (isTextInserted) {
                const insertedText = newText.substring(selectionStart, newPosition);
                const changeRangeLength = selectionEnd - selectionStart;
                if (changeRangeLength === 0) {
                    sharedString.insertText(selectionStart, insertedText);
                } else {
                    sharedString.replaceText(selectionStart, selectionEnd, insertedText);
                }
            } else {
                // Text was removed
                sharedString.removeText(newPosition, newPosition + charactersModifiedCount);
            }
        }
    }

    onUnmounted(() => {
      if (sharedString) {
        sharedString.off('sequenceDelta', sequenceDeltaChanged);
      }
    });

    return {
        textArea,
        text,
        updateSelection,
        handleChange
    }
  }
});
</script>
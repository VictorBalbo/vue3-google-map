<script lang="ts">
import { defineComponent, PropType, toRef, provide, computed, inject, markRaw, onBeforeUnmount, ref, watch } from "vue";
import { markerSymbol, apiSymbol, mapSymbol, markerClusterSymbol } from "../shared/index";
import equal from "fast-deep-equal";

const markerEvents = ["click", "drag", "dragend", "dragstart", "gmp-click"];
const intersectionObserver = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      entry.target.classList.add("drop");
      intersectionObserver.unobserve(entry.target);
    }
  }
});
export enum Animation {
  None = 0,
  Bounce = 1,
  Drop = 2,
}

export default defineComponent({
  name: "AdvancedMarker",
  props: {
    options: {
      type: Object as PropType<google.maps.marker.AdvancedMarkerElementOptions>,
      required: true,
    },
    pinOptions: {
      type: Object as PropType<google.maps.marker.PinElementOptions>,
      required: false,
    },
    animation: {
      type: Number as PropType<Animation>,
      required: false,
    },
  },
  emits: markerEvents,
  setup(props, { emit, expose, slots }) {
    const options = toRef(props, "options");
    const pinOptions = toRef(props, "pinOptions");
    const animation = toRef(props, "animation");

    const marker = ref<google.maps.marker.AdvancedMarkerElement>();

    const map = inject(mapSymbol, ref());
    const api = inject(apiSymbol, ref());
    const markerCluster = inject(markerClusterSymbol, ref());

    const isMarkerInCluster = computed(
      () => !!(markerCluster.value && api.value && marker.value instanceof google.maps.marker.AdvancedMarkerElement)
    );
    const handleAnimationChange = (animation?: Animation, oldAnimation?: Animation) => {
      const content = marker.value?.content as HTMLElement;
      // Recreating a pin shouldn't reanimate a pin drop
      if (animation === Animation.Drop && oldAnimation !== Animation.Drop) {
        content.classList.remove("bounce");
        content.addEventListener("animationend", () => content.classList.remove("drop"), { once: true });
        intersectionObserver.observe(content);
      }
      // Recreating a pin should continue animate a pin bounce
      if (animation === Animation.Bounce) {
        content.classList.remove("drop");
        content.classList.add("bounce");
      }
    };

    watch(
      [map, options, pinOptions, animation],
      async (_, [oldMap, oldOptions, oldPinOptions, oldAnimation]) => {
        const hasOptionChange =
          !equal(options.value, oldOptions) ||
          !equal(pinOptions.value, oldPinOptions) ||
          !equal(animation.value, oldAnimation);
        const hasChanged = hasOptionChange || map.value !== oldMap;

        if (!map.value || !api.value || !hasChanged) return;

        const { AdvancedMarkerElement, PinElement } = api.value.marker;

        if (marker.value) {
          const { map: _, content, ...otherOptions } = options.value;

          Object.assign(marker.value, {
            content: pinOptions.value ? new PinElement(pinOptions.value).element : content,
            ...otherOptions,
          });

          if (isMarkerInCluster.value) {
            markerCluster.value?.removeMarker(marker.value);
            markerCluster.value?.addMarker(marker.value);
          }
          handleAnimationChange(animation.value, oldAnimation);
        } else {
          if (pinOptions.value) {
            options.value.content = new PinElement(pinOptions.value).element;
          }

          marker.value = markRaw(new AdvancedMarkerElement(options.value));

          if (isMarkerInCluster.value) {
            markerCluster.value?.addMarker(marker.value);
          } else {
            marker.value.map = map.value;
          }

          markerEvents.forEach((event) => {
            marker.value?.addListener(event, (e: unknown) => emit(event, e));
          });
          handleAnimationChange(animation.value);
        }
      },
      {
        immediate: true,
      }
    );

    onBeforeUnmount(() => {
      if (marker.value) {
        api.value?.event.clearInstanceListeners(marker.value);

        if (isMarkerInCluster.value) {
          markerCluster.value?.removeMarker(marker.value);
        } else {
          marker.value.map = null;
        }
      }
    });

    provide(markerSymbol, marker);

    expose({ marker });

    return () => slots.default?.();
  },
});
</script>
<style lang="css">
@keyframes drop {
  0% {
    transform: translateY(-200px) scaleY(0.9);
    opacity: 0;
  }
  5% {
    opacity: 0.7;
  }
  50% {
    transform: translateY(0px) scaleY(1);
    opacity: 1;
  }
  65% {
    transform: translateY(-17px) scaleY(0.9);
    opacity: 1;
  }
  75% {
    transform: translateY(-22px) scaleY(0.9);
    opacity: 1;
  }
  100% {
    transform: translateY(0px) scaleY(1);
    opacity: 1;
  }
}
@keyframes bounce {
  0% {
    transform: translateY(0px);
    animation-timing-function: ease-out;
  }
  50% {
    transform: translateY(-20px);
    animation-timing-function: ease-in;
  }
  100% {
    transform: translateY(0px);
    animation-timing-function: ease-out;
  }
}
.drop {
  animation: drop 0.5s linear forwards;
}
.bounce {
  animation: bounce 1s linear infinite;
}
</style>

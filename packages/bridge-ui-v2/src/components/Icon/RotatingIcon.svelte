<script lang="ts">
  import Icon, { type IconType } from './Icon.svelte';
  export let loading: boolean;
  export let type: IconType;
  export let size: number = 20;
  export let viewBox: string;
</script>

<div class={loading ? 'rotating' : ''}>
  <Icon {type} {size} {viewBox} />
</div>

<style>
  .rotating {
    animation: rotation 2s infinite linear;
  }
  @keyframes rotation {
    from {
      transform: rotate(0deg);
    }
    to {
      transform: rotate(360deg);
    }
  }
</style>

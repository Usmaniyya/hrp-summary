<script>
  import { themes } from '../config.js'
  import { getContext } from 'svelte'

  export let theme = getContext('theme')
  export let filled = false
  export let center = true
</script>

<style>
  nav {
    font-family: 'Roboto Slab', serif;
    -webkit-box-sizing: border-box;
    box-sizing: border-box;
    position: relative;
    padding: 15px;
    border-bottom: 1px solid #777;
    z-index: 1;
  }
  nav h1 {
    font-size: 28px;
    color: #efb95c;
  }
</style>

<nav
  style="border-bottom-color: {themes[theme]['muted']}; {filled ? 'background-color: ' + themes[theme]['background'] + ';' : ''}">
  <div class="col-wide middle" class:center>
    <h1>NORTH-EAST NIGERIA (BORNO, ADAMAWA AND YOBE)</h1>
  </div>
</nav>

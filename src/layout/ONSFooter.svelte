<script>
  import { themes } from '../config.js'
  import { getContext } from 'svelte'

  export let theme = getContext('theme')
</script>

<style>
  footer {
    padding-bottom: 72px;
    padding-top: 36px;
    font-size: 18px;
  }
  footer a.link {
    -webkit-box-sizing: border-box;
    box-sizing: border-box;
    text-decoration: underline;
    display: inline-block;
  }
  footer a.link:hover {
    text-decoration: none;
  }
  footer a.link:active {
    outline: 3px solid transparent;
    background-color: #fd0;
    -webkit-box-shadow: 0 -2px #fd0, 0 4px #222;
    box-shadow: 0 -2px #fd0, 0 4px #222;
    color: #222;
    text-decoration: none;
  }
  footer hr {
    margin-bottom: 27px;
    margin-top: 36px;
    border: none;
    border-top: 1px solid #777;
  }
  ul {
    display: flex;
    flex-direction: column;
    gap: 8px;
    padding: 0;
    margin-top: 10px;
  }
  li {
    margin-right: 18px;
    display: inline-block;
  }
  .logo-img {
    margin-bottom: 27px;
  }
  .ogl-img {
    margin: 0 0.5rem 0.2rem 0;
    width: 59px;
    height: 24px;
    vertical-align: middle;
  }
  .license p {
    vertical-align: top;
    text-align: center;
  }
  a img:hover {
    cursor: pointer;
  }
</style>

<footer
  style="color: {themes[theme]['text']}; background-color: {themes[theme]['pale']};">
  <div class="col-wide" data-analytics="footer">
    <p class="footer_title">
      Get the latest updates, OCHA coordinates humanitarian action to ensure
      crisis-affected people receive the assistance and protection they need. It
      works to overcome obstacles that impede humanitarian assistance from
      reaching people affected by crises, and provides leadership in mobilizing
      assistance and resources on behalf of the humanitarian systems
    </p>
    <ul>
      <li>
        <a
          href="https://unocha.org/nigeria"
          class="link"
          target="_blank"
          style="color: #efb95c">
          www.unocha.org/nigeria
        </a>
      </li>
      <li>
        <a
          href="https://reports.unocha.org/en/country/nigeria"
          class="link"
          target="_blank"
          style="color: #efb95c">
          https://reports.unocha.org/en/country/nigeria
        </a>
      </li>
      <li>
        <a
          href="https://twitter.com/ochanigeria"
          class="link"
          target="_blank"
          style="color: #efb95c">
          twitter.com/ochanigeria
        </a>
      </li>
    </ul>
    <hr style="border-top-color: {themes[theme]['muted']}" />
    <div class="license">
      <p>Office for the Coordination of Humanitarian Affairs</p>
    </div>
  </div>
</footer>

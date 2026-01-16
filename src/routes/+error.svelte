<script>
  export let status;
  export let error;
</script>

<main class="error">
  <div class="card">
    <p class="code">{status}</p>
    <h1>Page not found</h1>
    <p class="message">{error?.message ?? "The page you are looking for does not exist."}</p>
    <a href="/">Go back home</a>
  </div>
</main>

<style>
  .error {
    min-height: 70vh;
    display: grid;
    place-items: center;
    padding: 3rem 1.5rem;
  }

  .card {
    background: white;
    border-radius: 1.25rem;
    padding: 2.5rem;
    text-align: center;
    box-shadow: 0 20px 45px rgba(15, 23, 42, 0.12);
    max-width: 420px;
    width: 100%;
  }

  .code {
    font-size: 3rem;
    font-weight: 700;
    color: #2b323b;
    margin: 0 0 0.5rem;
  }

  h1 {
    margin: 0 0 0.75rem;
    font-size: 1.6rem;
  }

  .message {
    margin: 0 0 1.5rem;
    color: #54616f;
  }

  a {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0.6rem 1.2rem;
    border-radius: 999px;
    background: #2b323b;
    color: white;
    text-decoration: none;
    font-weight: 500;
  }
</style>

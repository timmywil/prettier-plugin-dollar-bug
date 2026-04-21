<script>
  function format(n) {
    return '$' + n.toFixed(2);
  }

  let price = $state(19.99);
</script>

<p>{format(price)}</p>
<p>{`$${price.toFixed(2)}`}</p>

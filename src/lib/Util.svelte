<script content='module'>
  export function capitalizeFirstLetter(word) {
    let firstLetter = word.replace('_', ' ').charAt(0).toUpperCase();
    let remainingLetters = word.substring(1);
    return firstLetter + remainingLetters;
	}
</script>
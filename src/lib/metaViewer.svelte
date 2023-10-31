<script lang="ts">
	// import fileIcon from "../assets/file-solid.svg";

	let files;
	let results = [];
	let showImage = false; //Image flag
	let sortByName = false; //Sort flag by name

	function handleFilesChange() {
		if (files) {
			for (const file of files) {
				let dataUrlLoaded = false;
				let arrayBufferLoaded = false;
				let tempImageUrl;
				let tempMetaData;

				function checkAndAddResult() {
					// Add metadata and image to results
					if (dataUrlLoaded && arrayBufferLoaded) {
						const regexPattern =
							"parameters(.*?)Negative prompt:" +
							"(.*?)Steps:" +
							"(.*?), Sampler:" +
							"(.*?), CFG scale:" +
							"(.*?), Seed:" +
							"(.*?), Size:" +
							"(.*?), Model hash:" +
							"(.*?), Model:" +
							"(.*?), Version:" +
							"(.*?)$";
						const regex = new RegExp(regexPattern, "gs");
						let matches = regex.exec(tempMetaData);
						console.log(matches);

						const metaInfo = {
							prompt: matches[1],
							ngprompt: matches[2],
							steps: matches[3],
							sampler: matches[4],
							scale: matches[5],
							seed: matches[6],
							size: matches[7],
							hash: matches[8],
							model: matches[9],
							ver: matches[10],
						};
						results = [
							...results,
							{
								name: file.name,
								metaInfo: metaInfo,
								image: tempImageUrl,
								all: tempMetaData,
							},
						];
					}
				}

				// Get image
				const dataUrlReader = new FileReader();
				dataUrlReader.onload = function () {
					tempImageUrl = dataUrlReader.result;
					dataUrlLoaded = true;
					checkAndAddResult();
				};
				dataUrlReader.readAsDataURL(file);

				//Get metadata
				const arrayBufferReader = new FileReader();

				arrayBufferReader.onload = function () {
					const metadataChunks = parsePNGMetadata(
						arrayBufferReader.result
					);
					// Decode metadata
					const metadataStrings = metadataChunks.map((chunk) => {
						return new TextDecoder("utf-8").decode(chunk.data);
					});

					tempMetaData = metadataStrings.join(" ");
					arrayBufferLoaded = true;
					checkAndAddResult();
				};
				arrayBufferReader.readAsArrayBuffer(file);
			}
		}
	}

	function parsePNGMetadata(buffer) {
		const textDecoder = new TextDecoder("utf-8");
		const uint8Array = new Uint8Array(buffer);
		const chunks = [];

		let offset = 8; // PNG signature is 8 bytes
		while (offset < uint8Array.length) {
			const chunkLength =
				(uint8Array[offset] << 24) |
				(uint8Array[offset + 1] << 16) |
				(uint8Array[offset + 2] << 8) |
				uint8Array[offset + 3];
			const chunkType = textDecoder.decode(
				uint8Array.slice(offset + 4, offset + 8)
			);

			if (["tEXt", "zTXt", "iTXt"].includes(chunkType)) {
				const chunkData = uint8Array.slice(
					offset + 8,
					offset + 8 + chunkLength
				);
				chunks.push({ type: chunkType, data: chunkData });
			}

			offset += 12 + chunkLength;
		}
		return chunks;
	}

	function resetResults() {
		results = [];
		files = null;
	}

	$: sortedResults = sortByName
		? [...results].sort((a, b) => a.name.localeCompare(b.name))
		: [...results];
</script>

<div class="file_dropper">
	<label for="dropper">
		<svg xmlns="http://www.w3.org/2000/svg" height="1em" viewBox="0 0 384 512" class="fileicon"><!--! Font Awesome Free 6.4.2 by @fontawesome - https://fontawesome.com License - https://fontawesome.com/license (Commercial License) Copyright 2023 Fonticons, Inc. --><path d="M0 64C0 28.7 28.7 0 64 0H224V128c0 17.7 14.3 32 32 32H384V448c0 35.3-28.7 64-64 64H64c-35.3 0-64-28.7-64-64V64zm384 64H256V0L384 128z"/></svg><br>
		<div>Drop Generated PNG images here.</div>
		<input
			bind:files
			on:change={handleFilesChange}
			id="dropper"
			multiple
			type="file"
			accept="image/png"
		/>
	</label>
	
</div>

<div class="ctrl">
	<label for="show_img"
		><input
			type="checkbox"
			name="show_img"
			id="show_img"
			bind:checked={showImage}
		/>Show Images</label
	>
	<label for="sort_name"
		><input
			type="checkbox"
			name="sort_name"
			id="sort_name"
			bind:checked={sortByName}
		/>Sort by name</label
	>
	<button on:click={resetResults} class="reset">RESET</button>
</div>

{#if results.length > 0}
	<div class="results">
		{#each sortedResults as item, index}
			<div class="item info">
				<figure>
					{#if showImage}
						<img src={item.image} alt="" class="thumb" />
					{/if}
					<figcaption>{item.name}</figcaption>
				</figure>
				<dl class="prompt info">
					<dt>Prompt</dt>
					<dd>{item.metaInfo.prompt}</dd>
				</dl>
				<dl class="prompt info">
					<dt>Negative Prompt</dt>
					<dd>{item.metaInfo.ngprompt}</dd>
				</dl>
				<dl class="setting info">
					<dt>Setting</dt>
					<dd>Steps: {item.metaInfo.steps}</dd>
					<dd>Sampler: {item.metaInfo.sampler}</dd>
					<dd>Scale: {item.metaInfo.scale}</dd>
					<dd>Seed: {item.metaInfo.seed}</dd>
					<dd>Size: {item.metaInfo.size}</dd>
					<dd>Model: {item.metaInfo.model} ({item.metaInfo.hash})</dd>
					<dd>Ver: {item.metaInfo.ver}</dd>
				</dl>
			</div>
		{/each}
	</div>
{/if}

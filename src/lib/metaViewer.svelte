<script lang="ts">
	let files;
	let results = [];
	let showImage = false; //Image flag

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
</script>

<div class="file_dropper">
	<label for="dropper">
		Drop PNGs here.
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
		/>画像を表示</label
	>
	<button on:click={resetResults}>RESET</button>
</div>

{#if results.length > 0}
	<div class="results">
		{#each results as item, index}
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

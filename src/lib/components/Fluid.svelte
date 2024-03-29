<script>
	import { onMount } from 'svelte';
	import { fluidStore } from '$lib/stores/fluid.js';
	import { shaders, HSVtoRGB } from './webgl/helpers';

	let canvas;

	onMount(() => {
		/*
    Webgl effect inspired by https://github.com/PavelDoGreat/WebGL-Fluid-Simulation/blob/master/script.js

    MIT License
    Copyright (c) 2017 Pavel Dobryakov
    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:
    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.
    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.
    */
		let fluidStatus;
		fluidStore.subscribe((value) => {
			fluidStatus = value.status;
			if (fluidStatus) {
				window.addEventListener('mousemove', handleMouseMove);
			} else {
				window.removeEventListener('mousemove', handleMouseMove);
			}
		});

		resizeCanvas();

		let config = {
			SIM_RESOLUTION: 128,
			DYE_RESOLUTION: 512,
			CAPTURE_RESOLUTION: 512,
			DENSITY_DISSIPATION: 1,
			VELOCITY_DISSIPATION: 0.2,
			PRESSURE: 0.8,
			PRESSURE_ITERATIONS: 20,
			CURL: 30,
			SPLAT_RADIUS: 0.25,
			SPLAT_FORCE: 6000,
			SHADING: true,
			COLORFUL: true,
			COLOR_UPDATE_SPEED: 10,
			BACK_COLOR: { r: 0, g: 0, b: 0 }
		};

		function pointerPrototype() {
			this.id = -1;
			this.texcoordX = 0;
			this.texcoordY = 0;
			this.prevTexcoordX = 0;
			this.prevTexcoordY = 0;
			this.deltaX = 0;
			this.deltaY = 0;
			this.down = false;
			this.moved = false;
			this.color = [30, 0, 300];
		}

		let pointers = [];
		let splatStack = [];
		pointers.push(new pointerPrototype());

		const { gl, ext } = getWebGLContext(canvas);

		if (isMobile()) {
			config.DYE_RESOLUTION = 512;
		}
		if (!ext.supportLinearFiltering) {
			config.DYE_RESOLUTION = 512;
			config.SHADING = false;
		}

		function getWebGLContext(canvas) {
			const params = {
				alpha: true,
				depth: false,
				stencil: false,
				antialias: false,
				preserveDrawingBuffer: false
			};

			let gl = canvas.getContext('webgl2', params);
			const isWebGL2 = !!gl;
			if (!isWebGL2) gl = canvas.getContext('webgl', params) || canvas.getContext('experimental-webgl', params);

			let halfFloat;
			let supportLinearFiltering;
			if (isWebGL2) {
				gl.getExtension('EXT_color_buffer_float');
				supportLinearFiltering = gl.getExtension('OES_texture_float_linear');
			} else {
				halfFloat = gl.getExtension('OES_texture_half_float');
				supportLinearFiltering = gl.getExtension('OES_texture_half_float_linear');
			}

			gl.clearColor(0.0, 0.0, 0.0, 1.0);

			const halfFloatTexType = isWebGL2 ? gl.HALF_FLOAT : halfFloat.HALF_FLOAT_OES;
			let formatRGBA;
			let formatRG;
			let formatR;

			if (isWebGL2) {
				formatRGBA = getSupportedFormat(gl, gl.RGBA16F, gl.RGBA, halfFloatTexType);
				formatRG = getSupportedFormat(gl, gl.RG16F, gl.RG, halfFloatTexType);
				formatR = getSupportedFormat(gl, gl.R16F, gl.RED, halfFloatTexType);
			} else {
				formatRGBA = getSupportedFormat(gl, gl.RGBA, gl.RGBA, halfFloatTexType);
				formatRG = getSupportedFormat(gl, gl.RGBA, gl.RGBA, halfFloatTexType);
				formatR = getSupportedFormat(gl, gl.RGBA, gl.RGBA, halfFloatTexType);
			}

			return {
				gl,
				ext: {
					formatRGBA,
					formatRG,
					formatR,
					halfFloatTexType,
					supportLinearFiltering
				}
			};
		}

		function getSupportedFormat(gl, internalFormat, format, type) {
			if (!supportRenderTextureFormat(gl, internalFormat, format, type)) {
				switch (internalFormat) {
					case gl.R16F:
						return getSupportedFormat(gl, gl.RG16F, gl.RG, type);
					case gl.RG16F:
						return getSupportedFormat(gl, gl.RGBA16F, gl.RGBA, type);
					default:
						return null;
				}
			}

			return {
				internalFormat,
				format
			};
		}

		function supportRenderTextureFormat(gl, internalFormat, format, type) {
			let texture = gl.createTexture();
			gl.bindTexture(gl.TEXTURE_2D, texture);
			gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
			gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
			gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
			gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
			gl.texImage2D(gl.TEXTURE_2D, 0, internalFormat, 4, 4, 0, format, type, null);

			let fbo = gl.createFramebuffer();
			gl.bindFramebuffer(gl.FRAMEBUFFER, fbo);
			gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, texture, 0);

			let status = gl.checkFramebufferStatus(gl.FRAMEBUFFER);
			return status == gl.FRAMEBUFFER_COMPLETE;
		}

		function isMobile() {
			return /Android|iPhone/i.test(navigator.userAgent);
		}

		class Material {
			constructor(vertexShader, fragmentShaderSource) {
				this.vertexShader = vertexShader;
				this.fragmentShaderSource = fragmentShaderSource;
				this.programs = [];
				this.activeProgram = null;
				this.uniforms = [];
			}

			setKeywords(keywords) {
				let hash = 0;
				for (let i = 0; i < keywords.length; i++) hash += hashCode(keywords[i]);

				let program = this.programs[hash];
				if (program == null) {
					let fragmentShader = compileShader(gl.FRAGMENT_SHADER, this.fragmentShaderSource, keywords);
					program = createProgram(this.vertexShader, fragmentShader);
					this.programs[hash] = program;
				}

				if (program == this.activeProgram) return;

				this.uniforms = getUniforms(program);
				this.activeProgram = program;
			}

			bind() {
				gl.useProgram(this.activeProgram);
			}
		}

		class Program {
			constructor(vertexShader, fragmentShader) {
				this.uniforms = {};
				this.program = createProgram(vertexShader, fragmentShader);
				this.uniforms = getUniforms(this.program);
			}

			bind() {
				gl.useProgram(this.program);
			}
		}

		function createProgram(vertexShader, fragmentShader) {
			let program = gl.createProgram();
			gl.attachShader(program, vertexShader);
			gl.attachShader(program, fragmentShader);
			gl.linkProgram(program);

			if (!gl.getProgramParameter(program, gl.LINK_STATUS)) console.trace(gl.getProgramInfoLog(program));

			return program;
		}

		function getUniforms(program) {
			let uniforms = [];
			let uniformCount = gl.getProgramParameter(program, gl.ACTIVE_UNIFORMS);
			for (let i = 0; i < uniformCount; i++) {
				let uniformName = gl.getActiveUniform(program, i).name;
				uniforms[uniformName] = gl.getUniformLocation(program, uniformName);
			}
			return uniforms;
		}

		function compileShader(type, source, keywords) {
			source = addKeywords(source, keywords);

			const shader = gl.createShader(type);
			gl.shaderSource(shader, source);
			gl.compileShader(shader);

			if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) console.trace(gl.getShaderInfoLog(shader));

			return shader;
		}

		function addKeywords(source, keywords) {
			if (keywords == null) return source;
			let keywordsString = '';
			keywords.forEach((keyword) => {
				keywordsString += '#define ' + keyword + '\n';
			});
			return keywordsString + source;
		}

		const baseVertexShader = compileShader(gl.VERTEX_SHADER, shaders.baseVertexShaderSource);

		const blurVertexShader = compileShader(gl.VERTEX_SHADER, shaders.blurVertexShaderSource);

		const blurShader = compileShader(gl.FRAGMENT_SHADER, shaders.blurShaderSource);

		const copyShader = compileShader(gl.FRAGMENT_SHADER, shaders.copyShaderSource);

		const clearShader = compileShader(gl.FRAGMENT_SHADER, shaders.clearShaderSource);

		const colorShader = compileShader(gl.FRAGMENT_SHADER, shaders.colorShaderSource);

		const splatShader = compileShader(gl.FRAGMENT_SHADER, shaders.splatShaderSource);

		const advectionShader = compileShader(
			gl.FRAGMENT_SHADER,
			shaders.advectionShaderSource,
			ext.supportLinearFiltering ? null : ['MANUAL_FILTERING']
		);

		const divergenceShader = compileShader(gl.FRAGMENT_SHADER, shaders.divergenceShaderSource);

		const curlShader = compileShader(gl.FRAGMENT_SHADER, shaders.curlShaderSource);

		const vorticityShader = compileShader(gl.FRAGMENT_SHADER, shaders.vorticityShaderSource);

		const pressureShader = compileShader(gl.FRAGMENT_SHADER, shaders.pressureShaderSource);

		const gradientSubtractShader = compileShader(gl.FRAGMENT_SHADER, shaders.gradientSubtractShaderSource);

		const blit = (() => {
			gl.bindBuffer(gl.ARRAY_BUFFER, gl.createBuffer());
			gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([-1, -1, -1, 1, 1, 1, 1, -1]), gl.STATIC_DRAW);
			gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, gl.createBuffer());
			gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, new Uint16Array([0, 1, 2, 0, 2, 3]), gl.STATIC_DRAW);
			gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0);
			gl.enableVertexAttribArray(0);

			return (target, clear = false) => {
				if (target == null) {
					gl.viewport(0, 0, gl.drawingBufferWidth, gl.drawingBufferHeight);
					gl.bindFramebuffer(gl.FRAMEBUFFER, null);
				} else {
					gl.viewport(0, 0, target.width, target.height);
					gl.bindFramebuffer(gl.FRAMEBUFFER, target.fbo);
				}
				if (clear) {
					gl.clearColor(0.0, 0.0, 0.0, 1.0);
					gl.clear(gl.COLOR_BUFFER_BIT);
				}
				// CHECK_FRAMEBUFFER_STATUS();
				gl.drawElements(gl.TRIANGLES, 6, gl.UNSIGNED_SHORT, 0);
			};
		})();

		let dye;
		let velocity;
		let divergence;
		let curl;
		let pressure;

		// const blurProgram = new Program(blurVertexShader, blurShader);
		const copyProgram = new Program(baseVertexShader, copyShader);
		const clearProgram = new Program(baseVertexShader, clearShader);
		const colorProgram = new Program(baseVertexShader, colorShader);
		const splatProgram = new Program(baseVertexShader, splatShader);
		const advectionProgram = new Program(baseVertexShader, advectionShader);
		const divergenceProgram = new Program(baseVertexShader, divergenceShader);
		const curlProgram = new Program(baseVertexShader, curlShader);
		const vorticityProgram = new Program(baseVertexShader, vorticityShader);
		const pressureProgram = new Program(baseVertexShader, pressureShader);
		const gradienSubtractProgram = new Program(baseVertexShader, gradientSubtractShader);

		const displayMaterial = new Material(baseVertexShader, shaders.displayShaderSource);

		function initFramebuffers() {
			let simRes = getResolution(config.SIM_RESOLUTION);
			let dyeRes = getResolution(config.DYE_RESOLUTION);

			const texType = ext.halfFloatTexType;
			const rgba = ext.formatRGBA;
			const rg = ext.formatRG;
			const r = ext.formatR;
			const filtering = ext.supportLinearFiltering ? gl.LINEAR : gl.NEAREST;

			gl.disable(gl.BLEND);

			if (dye == null) dye = createDoubleFBO(dyeRes.width, dyeRes.height, rgba.internalFormat, rgba.format, texType, filtering);
			else dye = resizeDoubleFBO(dye, dyeRes.width, dyeRes.height, rgba.internalFormat, rgba.format, texType, filtering);

			if (velocity == null)
				velocity = createDoubleFBO(simRes.width, simRes.height, rg.internalFormat, rg.format, texType, filtering);
			else velocity = resizeDoubleFBO(velocity, simRes.width, simRes.height, rg.internalFormat, rg.format, texType, filtering);

			divergence = createFBO(simRes.width, simRes.height, r.internalFormat, r.format, texType, gl.NEAREST);
			curl = createFBO(simRes.width, simRes.height, r.internalFormat, r.format, texType, gl.NEAREST);
			pressure = createDoubleFBO(simRes.width, simRes.height, r.internalFormat, r.format, texType, gl.NEAREST);
		}

		function createFBO(w, h, internalFormat, format, type, param) {
			gl.activeTexture(gl.TEXTURE0);
			let texture = gl.createTexture();
			gl.bindTexture(gl.TEXTURE_2D, texture);
			gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, param);
			gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, param);
			gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
			gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
			gl.texImage2D(gl.TEXTURE_2D, 0, internalFormat, w, h, 0, format, type, null);

			let fbo = gl.createFramebuffer();
			gl.bindFramebuffer(gl.FRAMEBUFFER, fbo);
			gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, texture, 0);
			gl.viewport(0, 0, w, h);
			gl.clear(gl.COLOR_BUFFER_BIT);

			let texelSizeX = 1.0 / w;
			let texelSizeY = 1.0 / h;

			return {
				texture,
				fbo,
				width: w,
				height: h,
				texelSizeX,
				texelSizeY,
				attach(id) {
					gl.activeTexture(gl.TEXTURE0 + id);
					gl.bindTexture(gl.TEXTURE_2D, texture);
					return id;
				}
			};
		}

		function createDoubleFBO(w, h, internalFormat, format, type, param) {
			let fbo1 = createFBO(w, h, internalFormat, format, type, param);
			let fbo2 = createFBO(w, h, internalFormat, format, type, param);

			return {
				width: w,
				height: h,
				texelSizeX: fbo1.texelSizeX,
				texelSizeY: fbo1.texelSizeY,
				get read() {
					return fbo1;
				},
				set read(value) {
					fbo1 = value;
				},
				get write() {
					return fbo2;
				},
				set write(value) {
					fbo2 = value;
				},
				swap() {
					let temp = fbo1;
					fbo1 = fbo2;
					fbo2 = temp;
				}
			};
		}

		function resizeFBO(target, w, h, internalFormat, format, type, param) {
			let newFBO = createFBO(w, h, internalFormat, format, type, param);
			copyProgram.bind();
			gl.uniform1i(copyProgram.uniforms.uTexture, target.attach(0));
			blit(newFBO);
			return newFBO;
		}

		function resizeDoubleFBO(target, w, h, internalFormat, format, type, param) {
			if (target.width == w && target.height == h) return target;
			target.read = resizeFBO(target.read, w, h, internalFormat, format, type, param);
			target.write = createFBO(w, h, internalFormat, format, type, param);
			target.width = w;
			target.height = h;
			target.texelSizeX = 1.0 / w;
			target.texelSizeY = 1.0 / h;
			return target;
		}

		function updateKeywords() {
			let displayKeywords = [];
			if (config.SHADING) displayKeywords.push('SHADING');
			displayMaterial.setKeywords(displayKeywords);
		}

		updateKeywords();
		initFramebuffers();
		// multipleSplats(parseInt(Math.random() * 10) + 0);
		// multipleSplats(1);
		setTimeout(() => {
			multipleSplats(1);
		}, 500);

		let lastUpdateTime = Date.now();
		let colorUpdateTimer = 0.0;

		update();

		function update() {
			const dt = calcDeltaTime();
			if (resizeCanvas()) initFramebuffers();
			updateColors(dt);
			applyInputs();
			step(dt);
			render(null);
			requestAnimationFrame(update);
		}

		function calcDeltaTime() {
			let now = Date.now();
			let dt = (now - lastUpdateTime) / 1000;
			dt = Math.min(dt, 0.016666);
			lastUpdateTime = now;
			return dt;
		}

		function resizeCanvas() {
			let width = scaleByPixelRatio(canvas.clientWidth);
			let height = scaleByPixelRatio(canvas.clientHeight);
			if (canvas.width != width || canvas.height != height) {
				canvas.width = width;
				canvas.height = height;
				return true;
			}
			return false;
		}

		function updateColors(dt) {
			if (!config.COLORFUL) return;

			colorUpdateTimer += dt * config.COLOR_UPDATE_SPEED;
			if (colorUpdateTimer >= 1) {
				colorUpdateTimer = wrap(colorUpdateTimer, 0, 1);
				pointers.forEach((p) => {
					p.color = generateColor();
				});
			}
		}

		function applyInputs() {
			if (splatStack.length > 0) multipleSplats(splatStack.pop());

			pointers.forEach((p) => {
				if (p.moved) {
					p.moved = false;
					splatPointer(p);
				}
			});
		}

		function step(dt) {
			gl.disable(gl.BLEND);

			curlProgram.bind();
			gl.uniform2f(curlProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
			gl.uniform1i(curlProgram.uniforms.uVelocity, velocity.read.attach(0));
			blit(curl);

			vorticityProgram.bind();
			gl.uniform2f(vorticityProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
			gl.uniform1i(vorticityProgram.uniforms.uVelocity, velocity.read.attach(0));
			gl.uniform1i(vorticityProgram.uniforms.uCurl, curl.attach(1));
			gl.uniform1f(vorticityProgram.uniforms.curl, config.CURL);
			gl.uniform1f(vorticityProgram.uniforms.dt, dt);
			blit(velocity.write);
			velocity.swap();

			divergenceProgram.bind();
			gl.uniform2f(divergenceProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
			gl.uniform1i(divergenceProgram.uniforms.uVelocity, velocity.read.attach(0));
			blit(divergence);

			clearProgram.bind();
			gl.uniform1i(clearProgram.uniforms.uTexture, pressure.read.attach(0));
			gl.uniform1f(clearProgram.uniforms.value, config.PRESSURE);
			blit(pressure.write);
			pressure.swap();

			pressureProgram.bind();
			gl.uniform2f(pressureProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
			gl.uniform1i(pressureProgram.uniforms.uDivergence, divergence.attach(0));
			for (let i = 0; i < config.PRESSURE_ITERATIONS; i++) {
				gl.uniform1i(pressureProgram.uniforms.uPressure, pressure.read.attach(1));
				blit(pressure.write);
				pressure.swap();
			}

			gradienSubtractProgram.bind();
			gl.uniform2f(gradienSubtractProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
			gl.uniform1i(gradienSubtractProgram.uniforms.uPressure, pressure.read.attach(0));
			gl.uniform1i(gradienSubtractProgram.uniforms.uVelocity, velocity.read.attach(1));
			blit(velocity.write);
			velocity.swap();

			advectionProgram.bind();
			gl.uniform2f(advectionProgram.uniforms.texelSize, velocity.texelSizeX, velocity.texelSizeY);
			if (!ext.supportLinearFiltering)
				gl.uniform2f(advectionProgram.uniforms.dyeTexelSize, velocity.texelSizeX, velocity.texelSizeY);
			let velocityId = velocity.read.attach(0);
			gl.uniform1i(advectionProgram.uniforms.uVelocity, velocityId);
			gl.uniform1i(advectionProgram.uniforms.uSource, velocityId);
			gl.uniform1f(advectionProgram.uniforms.dt, dt);
			gl.uniform1f(advectionProgram.uniforms.dissipation, config.VELOCITY_DISSIPATION);
			blit(velocity.write);
			velocity.swap();

			if (!ext.supportLinearFiltering) gl.uniform2f(advectionProgram.uniforms.dyeTexelSize, dye.texelSizeX, dye.texelSizeY);
			gl.uniform1i(advectionProgram.uniforms.uVelocity, velocity.read.attach(0));
			gl.uniform1i(advectionProgram.uniforms.uSource, dye.read.attach(1));
			gl.uniform1f(advectionProgram.uniforms.dissipation, config.DENSITY_DISSIPATION);
			blit(dye.write);
			dye.swap();
		}

		function render(target) {
			gl.blendFunc(gl.ONE, gl.ONE_MINUS_SRC_ALPHA);
			gl.enable(gl.BLEND);

			drawColor(target, normalizeColor(config.BACK_COLOR));
			drawDisplay(target);
		}

		function drawColor(target, color) {
			colorProgram.bind();
			gl.uniform4f(colorProgram.uniforms.color, color.r, color.g, color.b, 1);
			blit(target);
		}

		function drawDisplay(target) {
			let width = target == null ? gl.drawingBufferWidth : target.width;
			let height = target == null ? gl.drawingBufferHeight : target.height;

			displayMaterial.bind();
			if (config.SHADING) gl.uniform2f(displayMaterial.uniforms.texelSize, 1.0 / width, 1.0 / height);
			gl.uniform1i(displayMaterial.uniforms.uTexture, dye.read.attach(0));
			blit(target);
		}

		function splatPointer(pointer) {
			let dx = pointer.deltaX * config.SPLAT_FORCE;
			let dy = pointer.deltaY * config.SPLAT_FORCE;
			splat(pointer.texcoordX, pointer.texcoordY, dx, dy, pointer.color);
		}

		function multipleSplats(amount) {
			for (let i = 0; i < amount; i++) {
				const color = generateColor();
				color.r *= 10.0;
				color.g *= 10.0;
				color.b *= 10.0;
				const x = Math.random();
				const y = Math.random();
				const dx = 1000 * (Math.random() - 0.5);
				const dy = 1000 * (Math.random() - 0.5);
				splat(x, y, dx, dy, color);
			}
		}

		function splat(x, y, dx, dy, color) {
			splatProgram.bind();
			gl.uniform1i(splatProgram.uniforms.uTarget, velocity.read.attach(0));
			gl.uniform1f(splatProgram.uniforms.aspectRatio, canvas.width / canvas.height);
			gl.uniform2f(splatProgram.uniforms.point, x, y);
			gl.uniform3f(splatProgram.uniforms.color, dx, dy, 0.0);
			gl.uniform1f(splatProgram.uniforms.radius, correctRadius(config.SPLAT_RADIUS / 100.0));
			blit(velocity.write);
			velocity.swap();

			gl.uniform1i(splatProgram.uniforms.uTarget, dye.read.attach(0));
			gl.uniform3f(splatProgram.uniforms.color, color.r, color.g, color.b);
			blit(dye.write);
			dye.swap();
		}

		function correctRadius(radius) {
			let aspectRatio = canvas.width / canvas.height;
			if (aspectRatio > 1) radius *= aspectRatio;
			return radius;
		}

		function handleMouseMove(e) {
			let pointer = pointers[0];
			let posX = scaleByPixelRatio(e.pageX);
			let posY = scaleByPixelRatio(e.pageY);
			updatePointerMoveData(pointer, posX, posY);
		}

		window.addEventListener('touchstart', (e) => {
			e.preventDefault();
			const touches = e.targetTouches;
			while (touches.length >= pointers.length) pointers.push(new pointerPrototype());
			for (let i = 0; i < touches.length; i++) {
				let posX = scaleByPixelRatio(touches[i].pageX);
				let posY = scaleByPixelRatio(touches[i].pageY);
				updatePointerDownData(pointers[i + 1], touches[i].identifier, posX, posY);
			}
		});

		window.addEventListener(
			'touchmove',
			(e) => {
				e.preventDefault();
				const touches = e.targetTouches;
				for (let i = 0; i < touches.length; i++) {
					let pointer = pointers[i + 1];
					if (!pointer.down) continue;
					let posX = scaleByPixelRatio(touches[i].pageX);
					let posY = scaleByPixelRatio(touches[i].pageY);
					updatePointerMoveData(pointer, posX, posY);
				}
			},
			false
		);

		window.addEventListener('touchend', (e) => {
			const touches = e.changedTouches;
			for (let i = 0; i < touches.length; i++) {
				let pointer = pointers.find((p) => p.id == touches[i].identifier);
				if (pointer == null) continue;
				updatePointerUpData(pointer);
			}
		});

		function updatePointerDownData(pointer, id, posX, posY) {
			pointer.id = id;
			pointer.down = true;
			pointer.moved = false;
			pointer.texcoordX = posX / canvas.width;
			pointer.texcoordY = 1.0 - posY / canvas.height;
			pointer.prevTexcoordX = pointer.texcoordX;
			pointer.prevTexcoordY = pointer.texcoordY;
			pointer.deltaX = 0;
			pointer.deltaY = 0;
			pointer.color = generateColor();
		}

		function updatePointerMoveData(pointer, posX, posY) {
			pointer.prevTexcoordX = pointer.texcoordX;
			pointer.prevTexcoordY = pointer.texcoordY;
			pointer.texcoordX = posX / canvas.width;
			pointer.texcoordY = 1.0 - posY / canvas.height;
			pointer.deltaX = correctDeltaX(pointer.texcoordX - pointer.prevTexcoordX);
			pointer.deltaY = correctDeltaY(pointer.texcoordY - pointer.prevTexcoordY);
			pointer.moved = Math.abs(pointer.deltaX) > 0 || Math.abs(pointer.deltaY) > 0;
		}

		function updatePointerUpData(pointer) {
			pointer.down = false;
		}

		function correctDeltaX(delta) {
			let aspectRatio = canvas.width / canvas.height;
			if (aspectRatio < 1) delta *= aspectRatio;
			return delta;
		}

		function correctDeltaY(delta) {
			let aspectRatio = canvas.width / canvas.height;
			if (aspectRatio > 1) delta /= aspectRatio;
			return delta;
		}

		function generateColor() {
			let c = HSVtoRGB(Math.random(), 1.0, 1.0);
			c.r *= 0.15;
			c.g *= 0.15;
			c.b *= 0.15;
			return c;
		}

		function normalizeColor(input) {
			let output = {
				r: input.r / 255,
				g: input.g / 255,
				b: input.b / 255
			};
			return output;
		}

		function wrap(value, min, max) {
			let range = max - min;
			if (range == 0) return min;
			return ((value - min) % range) + min;
		}

		function getResolution(resolution) {
			let aspectRatio = gl.drawingBufferWidth / gl.drawingBufferHeight;
			if (aspectRatio < 1) aspectRatio = 1.0 / aspectRatio;

			let min = Math.round(resolution);
			let max = Math.round(resolution * aspectRatio);

			if (gl.drawingBufferWidth > gl.drawingBufferHeight) return { width: max, height: min };
			else return { width: min, height: max };
		}

		function scaleByPixelRatio(input) {
			let pixelRatio = window.devicePixelRatio || 1;
			return Math.floor(input * pixelRatio);
		}

		function hashCode(s) {
			if (s.length == 0) return 0;
			let hash = 0;
			for (let i = 0; i < s.length; i++) {
				hash = (hash << 5) - hash + s.charCodeAt(i);
				hash |= 0;
			}
			return hash;
		}
	});
</script>

<canvas bind:this={canvas} class="pointer-events-none fixed bottom-0 left-0 right-0 top-0 -z-10 h-full w-full" />

<script>
    // https://stackoverflow.com/a/43960991

    class Colour {
        constructor(r, g, b) { this.set(r, g, b); }
        toString() { return `rgb(${Math.round(this.r)}, ${Math.round(this.g)}, ${Math.round(this.b)})`; }

        set(r, g, b) {
            this.r = this.clamp(r);
            this.g = this.clamp(g);
            this.b = this.clamp(b);
        }

        hueRotate(angle = 0) {
            angle = angle / 180 * Math.PI;
            let sin = Math.sin(angle);
            let cos = Math.cos(angle);

            this.multiply([
                0.213 + cos * 0.787 - sin * 0.213, 0.715 - cos * 0.715 - sin * 0.715, 0.072 - cos * 0.072 + sin * 0.928,
                0.213 - cos * 0.213 + sin * 0.143, 0.715 + cos * 0.285 + sin * 0.140, 0.072 - cos * 0.072 - sin * 0.283,
                0.213 - cos * 0.213 - sin * 0.787, 0.715 - cos * 0.715 + sin * 0.715, 0.072 + cos * 0.928 + sin * 0.072
            ]);
        }

        grayscale(value = 1) {
            this.multiply([
                0.2126 + 0.7874 * (1 - value), 0.7152 - 0.7152 * (1 - value), 0.0722 - 0.0722 * (1 - value),
                0.2126 - 0.2126 * (1 - value), 0.7152 + 0.2848 * (1 - value), 0.0722 - 0.0722 * (1 - value),
                0.2126 - 0.2126 * (1 - value), 0.7152 - 0.7152 * (1 - value), 0.0722 + 0.9278 * (1 - value)
            ]);
        }

        sepia(value = 1) {
            this.multiply([
                0.393 + 0.607 * (1 - value), 0.769 - 0.769 * (1 - value), 0.189 - 0.189 * (1 - value),
                0.349 - 0.349 * (1 - value), 0.686 + 0.314 * (1 - value), 0.168 - 0.168 * (1 - value),
                0.272 - 0.272 * (1 - value), 0.534 - 0.534 * (1 - value), 0.131 + 0.869 * (1 - value)
            ]);
        }

        saturate(value = 1) {
            this.multiply([
                0.213 + 0.787 * value, 0.715 - 0.715 * value, 0.072 - 0.072 * value,
                0.213 - 0.213 * value, 0.715 + 0.285 * value, 0.072 - 0.072 * value,
                0.213 - 0.213 * value, 0.715 - 0.715 * value, 0.072 + 0.928 * value
            ]);
        }

        multiply(matrix) {
            let newR = this.clamp(this.r * matrix[0] + this.g * matrix[1] + this.b * matrix[2]);
            let newG = this.clamp(this.r * matrix[3] + this.g * matrix[4] + this.b * matrix[5]);
            let newB = this.clamp(this.r * matrix[6] + this.g * matrix[7] + this.b * matrix[8]);
            this.r = newR; this.g = newG; this.b = newB;
        }

        brightness(value = 1) { this.linear(value); }
        contrast(value = 1) { this.linear(value, -(0.5 * value) + 0.5); }

        linear(slope = 1, intercept = 0) {
            this.r = this.clamp(this.r * slope + intercept * 255);
            this.g = this.clamp(this.g * slope + intercept * 255);
            this.b = this.clamp(this.b * slope + intercept * 255);
        }

        invert(value = 1) {
            this.r = this.clamp((value + (this.r / 255) * (1 - 2 * value)) * 255);
            this.g = this.clamp((value + (this.g / 255) * (1 - 2 * value)) * 255);
            this.b = this.clamp((value + (this.b / 255) * (1 - 2 * value)) * 255);
        }

        hsl() { // Code taken from https://stackoverflow.com/a/9493060/2688027, licensed under CC BY-SA.
            let r = this.r / 255;
            let g = this.g / 255;
            let b = this.b / 255;
            let max = Math.max(r, g, b);
            let min = Math.min(r, g, b);
            let h, s, l = (max + min) / 2;

            if(max === min) {
                h = s = 0;
            } else {
                let d = max - min;
                s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
                switch(max) {
                    case r: h = (g - b) / d + (g < b ? 6 : 0); break;
                    case g: h = (b - r) / d + 2; break;
                    case b: h = (r - g) / d + 4; break;
                } h /= 6;
            }

            return {
                h: h * 100,
                s: s * 100,
                l: l * 100
            };
        }

        clamp(value) {
            if(value > 255) { value = 255; }
            else if(value < 0) { value = 0; }
            return value;
        }
    }

    class Solver {
        constructor(target) {
            this.target = target;
            this.targetHSL = target.hsl();
            this.reusedColour = new Colour(0, 0, 0); // Object pool
        }

        solve() {
            let result = this.solveNarrow(this.solveWide());
            return {
                values: result.values,
                loss: result.loss,
                filter: this.css(result.values)
            };
        }

        solveWide() {
            const A = 5;
            const c = 15;
            const a = [60, 180, 18000, 600, 1.2, 1.2];

            let best = { loss: Infinity };
            for(let i = 0; best.loss > 25 && i < 3; i++) {
                let initial = [50, 20, 3750, 50, 100, 100];
                let result = this.spsa(A, a, c, initial, 1000);
                if(result.loss < best.loss) { best = result; }
            } return best;
        }

        solveNarrow(wide) {
            const A = wide.loss;
            const c = 2;
            const A1 = A + 1;
            const a = [0.25 * A1, 0.25 * A1, A1, 0.25 * A1, 0.2 * A1, 0.2 * A1];
            return this.spsa(A, a, c, wide.values, 500);
        }

        spsa(A, a, c, values, iters) {
            const alpha = 1;
            const gamma = 0.16666666666666666;

            let best = null;
            let bestLoss = Infinity;
            let deltas = new Array(6);
            let highArgs = new Array(6);
            let lowArgs = new Array(6);

            for(let k = 0; k < iters; k++) {
                let ck = c / Math.pow(k + 1, gamma);
                for(let i = 0; i < 6; i++) {
                    deltas[i] = Math.random() > 0.5 ? 1 : -1;
                    highArgs[i] = values[i] + ck * deltas[i];
                    lowArgs[i]  = values[i] - ck * deltas[i];
                }

                let lossDiff = this.loss(highArgs) - this.loss(lowArgs);
                for(let i = 0; i < 6; i++) {
                    let g = lossDiff / (2 * ck) * deltas[i];
                    let ak = a[i] / Math.pow(A + k + 1, alpha);
                    values[i] = fix(values[i] - ak * g, i);
                }

                let loss = this.loss(values);
                if(loss < bestLoss) { best = values.slice(0); bestLoss = loss; }
            } return { values: best, loss: bestLoss };

            function fix(value, idx) {
                let max = 100;
                if(idx === 2 /* saturate */) { max = 7500; }
                else if(idx === 4 /* brightness */ || idx === 5 /* contrast */) { max = 200; }

                if(idx === 3 /* hue-rotate */) {
                    if(value > max) { value = value % max; }
                    else if(value < 0) { value = max + value % max; }
                } else if(value < 0) { value = 0; }
                else if(value > max) { value = max; }
                return value;
            }
        }

        loss(filters) { // Argument is array of percentages.
            let colour = this.reusedColour;
            colour.set(0, 0, 0);

            colour.invert(filters[0] / 100);
            colour.sepia(filters[1] / 100);
            colour.saturate(filters[2] / 100);
            colour.hueRotate(filters[3] * 3.6);
            colour.brightness(filters[4] / 100);
            colour.contrast(filters[5] / 100);

            let colourHSL = colour.hsl();
            return Math.abs(colour.r - this.target.r)
                + Math.abs(colour.g - this.target.g)
                + Math.abs(colour.b - this.target.b)
                + Math.abs(colourHSL.h - this.targetHSL.h)
                + Math.abs(colourHSL.s - this.targetHSL.s)
                + Math.abs(colourHSL.l - this.targetHSL.l);
        }

        css(filters) {
            function fmt(idx, multiplier = 1) { return Math.round(filters[idx] * multiplier); }
            return `invert(${fmt(0)}%) sepia(${fmt(1)}%) saturate(${fmt(2)}%) hue-rotate(${fmt(3, 3.6)}deg) brightness(${fmt(4)}%) contrast(${fmt(5)}%)`;
        }
    }

    function hexToRgb(hex) {
        // Expand shorthand form (e.g. "03F") to full form (e.g. "0033FF")
        const shorthandRegex = /^#?([a-f\d])([a-f\d])([a-f\d])$/i;
        hex = hex.replace(shorthandRegex, (m, r, g, b) => {
            return r + r + g + g + b + b;
        });

        const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
        return result
            ? [
            parseInt(result[1], 16),
            parseInt(result[2], 16),
            parseInt(result[3], 16),
            ]
            : null;
    }

    let target = "#000000";
    let targetPixel;
    let resultPixel;
    let timeTaken = 0;
    let filter = "invert(0%)";

    function calculate() {
        console.log(hexToRgb(target))

        const colour = new Colour(...hexToRgb(target));
        const solver = new Solver(colour);
        const result = solver.solve();

        if (result.loss < 2) {
            calculate();
        } else {
            console.log(result.filter);
            filter = result.filter
        }
    }

    function calculateClicked() {
        const start = performance.now();
        calculate();
        timeTaken = performance.now() - start;
        
        targetPixel.style.backgroundColor = target;
        resultPixel.style.filter = filter;
    }

    function copyToClipboard() {
        navigator.clipboard.writeText(`filter: ${filter};`);
    }
</script>

<div class="container">
    <h3>CSS Filter Calculator</h3>
    <div id="controls">
        <input type="color" bind:value={target} />
        <input type="button" id="calculate" on:click={calculateClicked} value="Calculate" />
    </div>
    <div id="showcase">
        <div class="pixel-container">
            <h4 class="pixel-label">Target:</h4>
            <div class="pixel" id="target" bind:this={targetPixel}></div>
        </div>
        <div class="pixel-container">
            <h4 class="pixel-label">Result:</h4>
            <div class="pixel" id="result" bind:this={resultPixel}></div>
        </div>
    </div>
    <h3>Time taken: {timeTaken}ms</h3>
    <code>filter: {filter};</code>
    <input type="button" on:click={copyToClipboard} value="Copy to clipboard" />
</div>

<style>
    #controls {
        display: flex;
        align-items: center;
        gap: 1rem;        
    }

    #showcase {
        display: flex;
        flex-direction: column;
    }
    .pixel-container {
        display: flex;
        flex-direction: row;
        gap: 1rem;
    }
    .pixel-label {
        font-weight: 100;
    }

    .pixel {
        width: 4rem;
        height: 4rem;
        background-color: black;
    }

    code {
        border: 2px solid white;
        padding: .5rem;
        font-size: .8rem;
        width: fit-content;
    }
</style>

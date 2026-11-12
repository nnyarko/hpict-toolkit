# Third-party software licenses

The toolkit file `toolkit/HPI-CT_Toolkit.html` embeds the open-source libraries listed here so that it can run offline. Each stays under its own license. They are not covered by this project's MIT or CC BY 4.0 licenses, and their authors do not endorse this project. Names, versions and copyright lines were read from the license headers inside the toolkit file, except where noted.

## Libraries embedded directly

| Library | Version | License | Copyright line |
|---|---|---|---|
| Chart.js (https://www.chartjs.org) | 4.4.1 | MIT | (c) 2023 Chart.js Contributors |
| @kurkle/color (https://github.com/kurkle/color) | 0.3.2 | MIT | (c) 2023 Jukka Kurkela |
| Cytoscape.js (https://js.cytoscape.org) | 3.28.1 | MIT | Copyright (c) 2016-2023, The Cytoscape Consortium |

@kurkle/color is the color library that Chart.js bundles.

## MIT-licensed code bundled inside Cytoscape.js

Carrying its own header in the toolkit file:

| Component | License | Copyright line |
|---|---|---|
| Embeddable Minimum Strictly-Compliant Promises/A+ 1.1.1 Thenable | MIT | Copyright (c) 2013-2014 Ralf S. Engelschall (http://engelschall.com) |
| Bezier curve function generator | MIT | Copyright Gaetan Renaudeau |
| Runge-Kutta spring physics function generator, adapted from Framer.js | MIT | Copyright Koen Bok |

Bundled without a header of its own in the toolkit file (identified from the dependency list that Cytoscape.js 3.28.1 publishes, and from the code itself):

| Component | License | Copyright line |
|---|---|---|
| lodash (functions from the 4.17 series) | MIT | Copyright OpenJS Foundation and other contributors (https://openjsf.org/) |
| heap (0.2 series) | MIT | Xueqiao Xu |

The exact lodash and heap patch versions inside the Cytoscape.js build were not read from the file. Cytoscape.js 3.28.1 declares lodash ^4.17.21 and heap ^0.2.6.

## The MIT License

Every component above is released under the MIT License. Its text follows; read it with the copyright line of the component concerned.

```
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
```

## Trademark notice

CPT is a registered trademark of the American Medical Association. CPT code numbers are cited in this project for identification only, and no CPT descriptors are reproduced.

*See also [`THIRD_PARTY_RIGHTS.md`](THIRD_PARTY_RIGHTS.md) for standards and data sources, and [`ACKNOWLEDGEMENTS.md`](ACKNOWLEDGEMENTS.md) for credits.*

---
title: Matplotlib으로 Radar Chart 커스터마이징 하기
author: jisu
date: 2025-02-27 11:33:00 +0900
categories: [visualization]
tags: [matplotlib]
pin: false
math: true
mermaid: true
---

## 기본적인 matplotlib radar chart
일반적인 레이더 차트는 원형의 그리드로 표시되지만, 이를 커스터마이즈하여 다각형 그리드로 바꿀 수 있다.
그리드를 다각형으로 그리고 싶었던 나와 똑같은 고민을 한 흔적을 Stack Overflow에서 찾을 수 있었다.
> 출처: Stack Overflow - How to make radar/spider chart with pentagon grid using matplotlib and Python

일단, 원본 코드 그대로 가져왔다.

```python
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.patches import Circle, RegularPolygon
from matplotlib.path import Path
from matplotlib.projections.polar import PolarAxes
from matplotlib.projections import register_projection
from matplotlib.spines import Spine
from matplotlib.transforms import Affine2D


def radar_factory(num_vars, frame='circle'):
    """Create a radar chart with `num_vars` axes.

    This function creates a RadarAxes projection and registers it.

    Parameters
    ----------
    num_vars : int
        Number of variables for radar chart.
    frame : {'circle' | 'polygon'}
        Shape of frame surrounding axes.

    """
    # calculate evenly-spaced axis angles
    theta = np.linspace(0, 2*np.pi, num_vars, endpoint=False)

    class RadarTransform(PolarAxes.PolarTransform):
        def transform_path_non_affine(self, path):
            # Paths with non-unit interpolation steps correspond to gridlines,
            # in which case we force interpolation (to defeat PolarTransform's
            # autoconversion to circular arcs).
            if path._interpolation_steps > 1:
                path = path.interpolated(num_vars)
            return Path(self.transform(path.vertices), path.codes)

    class RadarAxes(PolarAxes):

        name = 'radar'
        PolarTransform = RadarTransform

        def __init__(self, *args, **kwargs):
            super().__init__(*args, **kwargs)
            # rotate plot such that the first axis is at the top
            self.set_theta_zero_location('N')

        def fill(self, *args, closed=True, **kwargs):
            """Override fill so that line is closed by default"""
            return super().fill(closed=closed, *args, **kwargs)

        def plot(self, *args, **kwargs):
            """Override plot so that line is closed by default"""
            lines = super().plot(*args, **kwargs)
            for line in lines:
                self._close_line(line)

        def _close_line(self, line):
            x, y = line.get_data()
            # FIXME: markers at x[0], y[0] get doubled-up
            if x[0] != x[-1]:
                x = np.concatenate((x, [x[0]]))
                y = np.concatenate((y, [y[0]]))
                line.set_data(x, y)

        def set_varlabels(self, labels):
            self.set_thetagrids(np.degrees(theta), labels)

        def _gen_axes_patch(self):
            # The Axes patch must be centered at (0.5, 0.5) and of radius 0.5
            # in axes coordinates.
            if frame == 'circle':
                return Circle((0.5, 0.5), 0.5)
            elif frame == 'polygon':
                return RegularPolygon((0.5, 0.5), num_vars, radius=0.5, edgecolor="k")
            else:
                raise ValueError("unknown value for 'frame': %s" % frame)

        def draw(self, renderer):
            """ Draw. If frame is polygon, make gridlines polygon-shaped """
            if frame == 'polygon':
                gridlines = self.yaxis.get_gridlines()
                for gl in gridlines:
                    gl.get_path()._interpolation_steps = num_vars
            super().draw(renderer)

        def _gen_axes_spines(self):
            if frame == 'circle':
                return super()._gen_axes_spines()
            elif frame == 'polygon':
                # spine_type must be 'left'/'right'/'top'/'bottom'/'circle'.
                spine = Spine(axes=self,
                              spine_type='circle',
                              path=Path.unit_regular_polygon(num_vars))
                # unit_regular_polygon gives a polygon of radius 1 centered at
                # (0, 0) but we want a polygon of radius 0.5 centered at (0.5,
                # 0.5) in axes coordinates.
                spine.set_transform(Affine2D().scale(.5).translate(.5, .5)
                                    + self.transAxes)
                return {'polar': spine}
            else:
                raise ValueError("unknown value for 'frame': %s" % frame)

    register_projection(RadarAxes)
    return theta


data = [['O1', 'O2', 'O3', 'O4', 'O5'],
        ('Title', [
                    [4, 3.5, 4, 2, 3,],
                    [1.07, 5.95, 2.04, 1.05, 0.00,],
                  ]
        )]

N = len(data[0])
theta = radar_factory(N, frame='polygon')

spoke_labels = data.pop(0)
title, case_data = data[0]
fig, ax = plt.subplots(figsize=(5, 5), subplot_kw=dict(projection='radar'))
fig.subplots_adjust(top=0.85, bottom=0.05)
ax.set_rgrids([0, 1, 2.0, 3.0, 4.0, 5.0, 6])
ax.set_title(title,  position=(0.5, 1.1), ha='center')

for d in case_data:
    line = ax.plot(theta, d)
    ax.fill(theta, d, alpha=0.25, label='_nolegend_')
ax.set_varlabels(spoke_labels)

plt.show()
```

위의 베이스 코드를 바탕으로 커스터마이징을 해보았다.
그래프에 한글이 나오게 하려면 한글 폰트를 따로 설치하거나 불러와야 한다.
폰트 경로를 설정하고 해당 폰트를 matplotlib에 적용해야한다.

## 그리드를 다각형으로 커스터마이징한 코드
```python
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.patches import Circle, RegularPolygon
from matplotlib.path import Path
from matplotlib.projections.polar import PolarAxes
from matplotlib.projections import register_projection
from matplotlib.spines import Spine
from matplotlib.transforms import Affine2D

# 레이더 차트 생성 함수
def radar_factory(num_vars, frame='circle'):
    """Create a radar chart with `num_vars` axes.

    This function creates a RadarAxes projection and registers it.

    Parameters
    ----------
    num_vars : int
        Number of variables for radar chart. (레이더 차트 축 수)
    frame : {'circle' | 'polygon'}
        Shape of frame surrounding axes. (축을 둘러싼 프레임 형태)

    """
    # 축 각도 계산 (변수 개수 만큼)
    theta = np.linspace(0, 2*np.pi, num_vars, endpoint=False)

    # 레이더 차트 변환 클래스 정의
    class RadarTransform(PolarAxes.PolarTransform):
        def transform_path_non_affine(self, path):
            # Paths with non-unit interpolation steps correspond to gridlines,
            # in which case we force interpolation (to defeat PolarTransform's
            # autoconversion to circular arcs).
            if path._interpolation_steps > 1:
                path = path.interpolated(num_vars)
            return Path(self.transform(path.vertices), path.codes)

    # 레이더 차트 Axes 클래스 정의
    class RadarAxes(PolarAxes):
        name = 'radar'
        PolarTransform = RadarTransform

        def __init__(self, *args, **kwargs):
            super().__init__(*args, **kwargs)
            # rotate plot such that the first axis is at the top
            self.set_theta_zero_location('N')

        def fill(self, *args, closed=True, **kwargs):
            """Override fill so that line is closed by default"""
            return super().fill(closed=closed, *args, **kwargs)

        def plot(self, *args, **kwargs):
            """Override plot so that line is closed by default"""
            lines = super().plot(*args, **kwargs)
            for line in lines:
                self._close_line(line)

        def _close_line(self, line):
            x, y = line.get_data()
            # FIXME: markers at x[0], y[0] get doubled-up
            if x[0] != x[-1]:
                x = np.concatenate((x, [x[0]]))
                y = np.concatenate((y, [y[0]]))
                line.set_data(x, y)

        def set_varlabels(self, labels):
            self.set_thetagrids(np.degrees(theta), labels)

        def _gen_axes_patch(self):
            # The Axes patch must be centered at (0.5, 0.5) and of radius 0.5
            # in axes coordinates.
            if frame == 'circle':
                return Circle((0.5, 0.5), 0.5)
            elif frame == 'polygon':
                return RegularPolygon((0.5, 0.5), num_vars, radius=0.5, edgecolor="k")
            else:
                raise ValueError("unknown value for 'frame': %s" % frame)

        # 프레임 그리드라인 그리기
        def draw(self, renderer):
            """ Draw. If frame is polygon, make gridlines polygon-shaped """
            if frame == 'polygon':
                gridlines = self.yaxis.get_gridlines()
                for gl in gridlines:
                    gl.get_path()._interpolation_steps = num_vars
            super().draw(renderer)

        def _gen_axes_spines(self):
            if frame == 'circle':
                return super()._gen_axes_spines()
            elif frame == 'polygon':
                # spine_type must be 'left'/'right'/'top'/'bottom'/'circle'.
                spine = Spine(axes=self,
                              spine_type='circle',
                              path=Path.unit_regular_polygon(num_vars))
                # unit_regular_polygon gives a polygon of radius 1 centered at
                # (0, 0) but we want a polygon of radius 0.5 centered at (0.5,
                # 0.5) in axes coordinates.
                spine.set_transform(Affine2D().scale(.5).translate(.5, .5)
                                    + self.transAxes)
                return {'polar': spine}
            else:
                raise ValueError("unknown value for 'frame': %s" % frame)

    register_projection(RadarAxes)
    return theta

# 한글 폰트 설치
# !apt-get -qq install -y fonts-nanum
# 한글 폰트 설정!!!!!!!!!!!!
# import matplotlib.font_manager as fm
# font_path = '경로 지정!!!!!!!!!!'
# font_prop = fm.FontProperties(fname=font_path)
# font_prop.get_name()

# 데이터 준비
data = [['O1', 'O2', 'O3', 'O4', 'O5'],
        ('Title', [
                    [4, 3.5, 4, 2, 3,],
                    [1.07, 5.95, 2.04, 1.05, 0.00,],
                  ]
        )]

# 축 개수 (5개)
N = len(data[0])

# 다각형 레이더 차트 생성
theta = radar_factory(N, frame='polygon')

# 차트의 축 레이블
spoke_labels = data.pop(0)
title, case_data = data[0]
fig, ax = plt.subplots(figsize=(5, 5), subplot_kw=dict(projection='radar'))
fig.subplots_adjust(top=0.85, bottom=0.05)
ax.set_rgrids([])

# 제목
ax.set_title(title,  position=(0.5, 1.1), ha='center', pad=40)

# 색상
colors = ['#FF9999', '#66B2FF']

# 데이터 플로팅
labels = ['Apple', 'Samsung']
for i, d in enumerate(case_data):
    color = colors[i % len(colors)]  # 색상 지정
    label = labels[i]
    line = ax.plot(theta, d, color=color, label=label)  # 레이블을 label로 설정
    ax.fill(theta, d, color=color, alpha=0.25)

# 축 레이블
ax.set_varlabels(spoke_labels)

# 범례 추가
ax.legend(loc='upper right', bbox_to_anchor=(1.3, 1.1), title='Legend')

plt.show()
```

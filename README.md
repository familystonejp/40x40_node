import sys
import cv2
import numpy as np
from PyQt5.QtWidgets import QApplication, QLabel, QSlider, QVBoxLayout, QWidget
from PyQt5.QtCore import Qt, QTimer
from matplotlib.backends.backend_qt5agg import FigureCanvasQTAgg as FigureCanvas
from matplotlib.figure import Figure

class BrightnessPlot(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("リアルタイム明るさ波形")
        self.resize(800, 600)

        self.brightness_data = [0] * 400
        self.offset = 0

        self.figure = Figure()
        self.canvas = FigureCanvas(self.figure)
        self.ax = self.figure.add_subplot(111)
        self.line, = self.ax.plot(self.brightness_data, lw=2)
        self.ax.axvline(x=len(self.brightness_data) // 2, color='r', linestyle='--')  # センターマーカー
        self.ax.set_ylim(0, 255)
        self.ax.set_xlim(0, len(self.brightness_data))

        self.slider = QSlider(Qt.Horizontal)
        self.slider.setMinimum(-200)
        self.slider.setMaximum(200)
        self.slider.setValue(0)
        self.slider.valueChanged.connect(self.update_offset)

        layout = QVBoxLayout()
        layout.addWidget(self.canvas)
        layout.addWidget(QLabel("波形オフセット"))
        layout.addWidget(self.slider)
        self.setLayout(layout)

        self.cap = cv2.VideoCapture(0)
        self.timer = QTimer()
        self.timer.timeout.connect(self.update_plot)
        self.timer.start(30)

    def update_offset(self, value):
        self.offset = value

    def update_plot(self):
        ret, frame = self.cap.read()
        if not ret:
            return

        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        h, w = gray.shape
        center_y, center_x = h // 2, w // 2
        roi = gray[center_y-2:center_y+3, center_x-2:center_x+3]
        avg_brightness = int(np.mean(roi))

        self.brightness_data.append(avg_brightness)
        self.brightness_data.pop(0)

        shifted_data = np.roll(self.brightness_data, self.offset)
        self.line.set_ydata(shifted_data)
        self.canvas.draw()

    def closeEvent(self, event):
        self.cap.release()
        event.accept()

if __name__ == '__main__':
    app = QApplication(sys.argv)
    window = BrightnessPlot()
    window.show()
    sys.exit(app.exec_())

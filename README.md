# SingleHandMode（单手模式）
基于Android M，修改系统源码实现单手模式。
--------------------------------------------------------------------------------------------------------------------------------------

ANDROID从版本4.2开始提供了一个显示管理服务DisplayManagerService,支持多种显示类型的多个显示器的镜像显示，包括内建的显示类型（本地）、HDMI显示类型以及支持WIFI Display 协议( MIRACAST)，实现本地设备在远程显示器上的镜像显示。
--------------------------------------------------------------------------------------------------------------------------------------

修改DMS,在其中添加一个触发单手模式的方法 void singleHandModeChange(boolean change,boolean left).

diff --git a/core/java/android/hardware/display/DisplayManager.java b/core/java/android/hardware/display/DisplayManager.java
index 80f8fd3..6a49816 100644
--- a/core/java/android/hardware/display/DisplayManager.java
+++ b/core/java/android/hardware/display/DisplayManager.java
@@ -543,6 +543,13 @@ public final class DisplayManager {
                 name, width, height, densityDpi, surface, flags, callback, handler);
     }
 
+    // single hand mode
+    /** @hide */
+    public void singleHandModeChange(boolean change, boolean left) {
+        mGlobal.singleHandModeChange(change, left);
+    }
     /**
      * Get the plug status of SmartBook
      *
diff --git a/core/java/android/hardware/display/DisplayManagerGlobal.java b/core/java/android/hardware/display/DisplayManagerGlobal.java
index 3869992..62a98e9 100644
--- a/core/java/android/hardware/display/DisplayManagerGlobal.java
+++ b/core/java/android/hardware/display/DisplayManagerGlobal.java
@@ -427,6 +427,17 @@ public final class DisplayManagerGlobal {
         }
     }
 
+    // single hand mode
+    public void singleHandModeChange(boolean change, boolean left) {
+        try {
+            mDm.singleHandModeChange(change, left);
+        } catch (RemoteException ex) {
+            Log.w("SingleHandMode", "SingleHandMode Failed to singleHandChange error:" + ex.toString());
+        }
+    }
+    
     public boolean isSinkEnabled() {
         boolean enabled = false;
         try {
diff --git a/core/java/android/hardware/display/IDisplayManager.aidl b/core/java/android/hardware/display/IDisplayManager.aidl
index 89e386d..afe90f2 100644
--- a/core/java/android/hardware/display/IDisplayManager.aidl
+++ b/core/java/android/hardware/display/IDisplayManager.aidl
@@ -84,4 +84,9 @@ interface IDisplayManager {
     void suspendWifiDisplay(boolean suspend, in Surface surface);
 
     void sendUibcInputEvent(String input);
+
+    // single hand mode
+    void singleHandModeChange(boolean change,boolean left);
 }

diff --git a/services/core/java/com/android/server/display/DisplayManagerService.java b/services/core/java/com/android/server/display/DisplayManagerService.java
index 81e8c54..7371d6f 100755
--- a/services/core/java/com/android/server/display/DisplayManagerService.java
+++ b/services/core/java/com/android/server/display/DisplayManagerService.java
@@ -155,6 +155,12 @@ public final class DisplayManagerService extends SystemService {
     // services should be started.  This option may disable certain display adapters.
     public boolean mOnlyCore;
 
+    // single hand mode
+    private boolean mSingleHandModeState = false;
+    private boolean mSingleHandModeLeft = true;
+
     // True if the display manager service should pretend there is only one display
     // and only tell applications about the existence of the default logical display.
     // The display manager can still mirror content to secondary displays but applications
@@ -944,6 +950,13 @@ public final class DisplayManagerService extends SystemService {
                     + device.getDisplayDeviceInfoLocked());
             return;
         }
+
+        // single hand mode
+        display.mSingleHandModeState = mSingleHandModeState;
+        display.mSingleHandModeLeft = mSingleHandModeLeft;
+
         display.configureDisplayInTransactionLocked(device, info.state == Display.STATE_OFF);
 
         // Update the viewports if needed.
@@ -1255,6 +1268,17 @@ public final class DisplayManagerService extends SystemService {
             }
         }
 
+        // single hand mode
+        public void singleHandModeChange(boolean change, boolean left) {
+            mSingleHandModeState = change;
+            mSingleHandModeLeft = left;
+            Slog.d("SingleHandMode", "singleHandMode  singleHandMode singleHandModeState=" + mSingleHandModeState + " left " + mSingleHandModeLeft);
+            scheduleTraversalLocked(false);
+            performTraversalInTransactionFromWindowManagerInternal();
+        }
+
         /**
          * Returns the list of all display ids.
          */
diff --git a/services/core/java/com/android/server/display/DisplayPowerController.java b/services/core/java/com/android/server/display/DisplayPowerController.java
index c42b8fb..63a5fd3 100755
--- a/services/core/java/com/android/server/display/DisplayPowerController.java
+++ b/services/core/java/com/android/server/display/DisplayPowerController.java
@@ -704,6 +704,14 @@ final class DisplayPowerController implements AutomaticBrightnessController.Call
             state = Display.STATE_OFF;
         }
 
+        // single hand mode
+        if (state == Display.STATE_OFF) {
+            mWindowManagerPolicy.closeSingleHandMode();
+            Slog.d("SingleHandMode", "DisplayPowerController---closeSingleHandMode---state : " + state);
+        }
+
         /// M: dismiss ColorFade when IPO boot up
         final boolean ipoShutdown = mIPOShutDown;
 
diff --git a/services/core/java/com/android/server/display/LogicalDisplay.java b/services/core/java/com/android/server/display/LogicalDisplay.java
index 4b89ef5..14e9d7b 100644
--- a/services/core/java/com/android/server/display/LogicalDisplay.java
+++ b/services/core/java/com/android/server/display/LogicalDisplay.java
@@ -22,6 +22,7 @@
 package com.android.server.display;
 
 import android.graphics.Rect;
+import android.os.SystemProperties;
 import android.view.Display;
 import android.view.DisplayInfo;
 import android.view.Surface;
@@ -77,6 +78,13 @@ final class LogicalDisplay {
     private DisplayDevice mPrimaryDisplayDevice;
     private DisplayDeviceInfo mPrimaryDisplayDeviceInfo;
 
+    // single hand mode
+    private static final String SINGLE_HAND_MODE_SUPPORTED = "ro.android.singlehandmode";
+    public boolean mSingleHandModeState = false;
+    public boolean mSingleHandModeLeft = true;
+
     // True if the logical display has unique content.
     private boolean mHasContent;
 
@@ -349,8 +357,45 @@ final class LogicalDisplay {
 
         int displayRectTop = (physHeight - displayRectHeight) / 2;
         int displayRectLeft = (physWidth - displayRectWidth) / 2;
-        mTempDisplayRect.set(displayRectLeft, displayRectTop,
-                displayRectLeft + displayRectWidth, displayRectTop + displayRectHeight);
+        /*mTempDisplayRect.set(displayRectLeft, displayRectTop,
+                displayRectLeft + displayRectWidth, displayRectTop + displayRectHeight);*/
+        // single hand mode
+        final boolean singleHandModeSupported = SystemProperties.get(SINGLE_HAND_MODE_SUPPORTED, "false").equals("true");
+        if (singleHandModeSupported) {
+            int displayRectRight = displayRectLeft + displayRectWidth;
+            int displayRectBottom = displayRectTop + displayRectHeight;
+
+            if ((mSingleHandModeState == true) && (!isBlanked)) {
+                if (mSingleHandModeLeft == true) {
+                    displayRectTop = displayRectTop + displayRectHeight / 4;
+                    displayRectRight = displayRectLeft + displayRectWidth * 3 / 4;
+                } else {
+                    displayRectLeft = displayRectLeft + displayRectWidth / 4;
+                    displayRectTop = displayRectTop + displayRectHeight / 4;
+                }
+            }
+
+            mTempDisplayRect.set(displayRectLeft, displayRectTop, displayRectRight, displayRectBottom);
+        } else {
+            mTempDisplayRect.set(displayRectLeft, displayRectTop,
+                    displayRectLeft + displayRectWidth, displayRectTop + displayRectHeight);
+        }
 
         mTempDisplayRect.left += mDisplayOffsetX;
         mTempDisplayRect.right += mDisplayOffsetX;
--------------------------------------------------------------------------------------------------------------------------------------
真正单手模式的改变屏幕的大小是在LogicalDisplay.java中，通过修改mTempDisplayRect实现显示屏幕的的改变。

--------------------------------------------------------------------------------------------------------------------------------------
#SingleHandMode（单手模式）的触发方式
通过快滑动依次点击Home按键和Back按键启动单手模式（屏幕缩小到左下角，屏幕宽高变为原始屏幕的3/4）。此时再重复上述手势，则退出单手模式。
通过快滑动依次点击Home按键和Menu按键启动单手模式（屏幕缩小到右下角，屏幕宽高变为原始屏幕的3/4）。此时再重复上述手势，则退出单手模式

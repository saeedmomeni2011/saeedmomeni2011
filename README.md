//+------------------------------------------------------------------+
//|                                                  ChartSaver.mq4 |
//|                        Copyright 2023, MetaQuotes Software Corp. |
//|                                             https://www.mql5.com |
//+------------------------------------------------------------------+
#property copyright "Copyright 2023"
#property version   "1.00"
#property strict
#property indicator_chart_window

//+------------------------------------------------------------------+
//| Custom indicator initialization function                         |
//+------------------------------------------------------------------+
int OnInit()
  {
//--- ایجاد دکمه روی چارت
   CreateSaveButton();
//---
   return(INIT_SUCCEEDED);
  }

//+------------------------------------------------------------------+
//| ایجاد دکمه ذخیره سازی                                           |
//+------------------------------------------------------------------+
void CreateSaveButton()
  {
//--- تنظیمات دکمه
   string btn_name = "SaveChartBtn";
   int x_distance = 10;    // فاصله افقی از گوشه چپ
   int y_distance = 10;    // فاصله عمودی از بالا
   int width = 100;        // عرض دکمه
   int height = 30;        // ارتفاع دکمه

//--- ایجاد دکمه
   ObjectCreate(0, btn_name, OBJ_BUTTON, 0, 0, 0);
   ObjectSetInteger(0, btn_name, OBJPROP_XDISTANCE, x_distance);
   ObjectSetInteger(0, btn_name, OBJPROP_YDISTANCE, y_distance);
   ObjectSetInteger(0, btn_name, OBJPROP_XSIZE, width);
   ObjectSetInteger(0, btn_name, OBJPROP_YSIZE, height);
   ObjectSetString(0, btn_name, OBJPROP_TEXT, "Save Chart");
   ObjectSetInteger(0, btn_name, OBJPROP_COLOR, clrWhite);
   ObjectSetInteger(0, btn_name, OBJPROP_BGCOLOR, clrRoyalBlue);
   ObjectSetInteger(0, btn_name, OBJPROP_BORDER_COLOR, clrNavy);
  }

//+------------------------------------------------------------------+
//| تابع رویداد چارت                                                |
//+------------------------------------------------------------------+
void OnChartEvent(const int id,
                  const long &lparam,
                  const double &dparam,
                  const string &sparam)
  {
//--- اگر رویداد کلیک روی دکمه باشد
   if(id == CHARTEVENT_OBJECT_CLICK)
     {
      if(sparam == "SaveChartBtn")
        {
         SaveChartAsImage();
        }
     }
  }

//+------------------------------------------------------------------+
//| ذخیره چارت به عنوان عکس                                         |
//+------------------------------------------------------------------+
void SaveChartAsImage()
  {
//--- تنظیم نام فایل
   string filename = "Chart_" + Symbol() + "_" + TimeToString(TimeCurrent(), TIME_DATE|TIME_MINUTES);
   StringReplace(filename, ".", "_");
   StringReplace(filename, ":", "_");

//--- ذخیره عکس با سایز مشخص
   ChartScreenShot(0, filename + ".png", 1526, 896, ALIGN_CENTER);
   Alert("Chart saved as: ", filename);
  }

//+------------------------------------------------------------------+
//| Custom indicator deinitialization function                       |
//+------------------------------------------------------------------+
int OnDeinit(const int reason)
  {
//--- حذف دکمه هنگام حذف اندیکاتور
   ObjectDelete("SaveChartBtn");
//---
   return(0);
  }
//+------------------------------------------------------------------+

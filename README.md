# -RiskToRewardPro
//+------------------------------------------------------------------+
//|                                               RiskToRewardPro.mq5 |
//|                                 Copyright 2024, KralFx & Copilot |
//|                                              https://www.mql5.com |
//+------------------------------------------------------------------+
#property copyright "KralFx & Copilot"
#property link      "https://www.mql5.com"
#property version   "1.00"
#property description "اندیکاتور پیشرفته مدیریت ریسک و تعیین سطوح معاملاتی"
#property indicator_chart_window
#property strict

//+------------------------------------------------------------------+
//| تنظیمات ورودی                                                   |
//+------------------------------------------------------------------+
input double   InpEntryPrice    = 0.0;       // قیمت ورود (0=قیمت فعلی)
input double   InpStopLoss      = 0.0;       // استاپ لاس (0=اتوماتیک)
input double   InpRiskReward    = 2.0;       // نسبت ریسک به ریوارد
input double   InpLotSize       = 0.01;      // حجم معامله
input double   InpFixedRiskPips = 0;         // ریسک ثابت (پیپ)

input bool     InpEnableTP2     = true;      // فعال‌سازی TP2
input bool     InpEnableTP3     = false;     // فعال‌سازی TP3
input double   InpTP2Multiplier = 2.0;       // ضریب TP2
input double   InpTP3Multiplier = 3.0;       // ضریب TP3

input color    InpEntryColor    = clrLime;   // رنگ خط ورود
input color    InpSLColor       = clrRed;    // رنگ استاپ لاس
input color    InpTP1Color      = clrBlue;   // رنگ TP1
input color    InpTP2Color      = clrGold;   // رنگ TP2
input color    InpTP3Color      = clrMagenta;// رنگ TP3
input int      InpLineWidth     = 2;         // ضخامت خطوط
input bool     InpShowLabels    = true;      // نمایش لیبل‌ها
input bool     InpShowInfo      = true;      // نمایش اطلاعات

input color    InpRiskColor     = clrRed;    // رنگ ناحیه ریسک
input color    InpRewardColor   = clrGreen;  // رنگ ناحیه ریوارد
input bool     InpShowRiskArea  = true;      // نمایش ناحیه ریسک
input bool     InpShowRewardArea= true;      // نمایش ناحیه ریوارد
input int      InpAreaAlpha     = 60;        // شفافیت نواحی

input bool     InpAlertOnTouch  = true;      // هشدار برخورد به سطوح
input bool     InpPlaySound     = true;      // پخش صدا
input string   InpSoundFile     = "alert.wav";// فایل صوتی

//+------------------------------------------------------------------+
//| متغیرهای سراسری                                                 |
//+------------------------------------------------------------------+
double   entryPrice, slPrice, tp1Price, tp2Price, tp3Price;
bool     isLong;
string   objPrefix = "RR_";
int      clickCount = 0;
double   price1 = 0, price2 = 0;
datetime lastAlertTime = 0;

//+------------------------------------------------------------------+
//| تابع مقداردهی اولیه                                            |
//+------------------------------------------------------------------+
int OnInit()
{
   Print("=== RiskToReward Pro Indicator Started ===");
   
   // تنظیم مقادیر اولیه
   if(InpEntryPrice == 0.0)
      entryPrice = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   else
      entryPrice = InpEntryPrice;
      
   if(InpStopLoss == 0.0)
      slPrice = entryPrice - 100 * _Point;
   else
      slPrice = InpStopLoss;
   
   isLong = (slPrice < entryPrice);
   
   // محاسبه سطوح Take Profit
   CalculateTPLevels();
   
   // رسم تمام المان‌ها
   DrawAllObjects();
   UpdateInfoPanel();
   
   Comment("🎯 RiskToReward Pro فعال شد\n" +
           "💡 راهنما: دو بار کلیک برای تنظیم سطوح | R=ریست | C=پاک کردن");
           
   return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| تابع اصلی هر تیک                                                |
//+------------------------------------------------------------------+
void OnTick()
{
   // بروزرسانی پنل اطلاعات
   if(InpShowInfo)
      UpdateInfoPanel();
   
   // بررسی هشدارها
   if(InpAlertOnTouch)
      CheckAlerts();
}

//+------------------------------------------------------------------+
//| مدیریت رویدادهای چارت                                           |
//+------------------------------------------------------------------+
void OnChartEvent(const int id, const long &lparam, const double &dparam, const string &sparam)
{
   // مدیریت کلیک ماوس
   if(id == 1) // CHARTEVENT_CLICK
   {
      HandleClickEvent(lparam, dparam);
   }
   // مدیریت فشار کلید
   else if(id == 7) // CHARTEVENT_KEYDOWN
   {
      HandleKeyEvent(lparam);
   }
   // مدیریت کشیدن آبجکت
   else if(id == 5) // CHARTEVENT_OBJECT_DRAG
   {
      HandleDragEvent(sparam);
   }
   // مدیریت پایان کشیدن آبجکت
   else if(id == 6) // CHARTEVENT_OBJECT_END_EDIT
   {
      RefreshAll();
   }
}

//+------------------------------------------------------------------+
//| مدیریت کلیک ماوس                                                |
//+------------------------------------------------------------------+
void HandleClickEvent(long lparam, double dparam)
{
   int x = (int)lparam;
   int y = (int)dparam;
   int subwindow;
   datetime time;
   double price;
   
   if(ChartXYToTimePrice(0, x, y, subwindow, time, price))
   {
      if(subwindow == 0) // فقط در پنجره اصلی چارت
      {
         clickCount++;
         
         if(clickCount == 1)
         {
            price1 = price;
            Print("📍 کلیک اول در قیمت: ", DoubleToString(price1, _Digits));
         }
         else if(clickCount == 2)
         {
            price2 = price;
            Print("📍 کلیک دوم در قیمت: ", DoubleToString(price2, _Digits));
            ProcessTwoClicks(price1, price2);
            clickCount = 0;
         }
      }
   }
}

//+------------------------------------------------------------------+
//| پردازش دو کلیک متوالی                                           |
//+------------------------------------------------------------------+
void ProcessTwoClicks(double p1, double p2)
{
   if(p1 < p2)
   {
      // پوزیشن LONG
      slPrice = p1;
      entryPrice = p2;
      isLong = true;
      Print("📈 پوزیشن LONG تنظیم شد");
   }
   else
   {
      // پوزیشن SHORT
      slPrice = p2;
      entryPrice = p1;
      isLong = false;
      Print("📉 پوزیشن SHORT تنظیم شد");
   }
   
   RefreshAll();
}

//+------------------------------------------------------------------+
//| مدیریت فشار کلید                                                |
//+------------------------------------------------------------------+
void HandleKeyEvent(long keycode)
{
   if(keycode == 82) // کلید R
   {
      ResetIndicator();
      Print("🔁 اندیکاتور ریست شد");
   }
   else if(keycode == 67) // کلید C
   {
      DeleteAllObjects();
      Print("🧹 تمام آبجکت‌ها پاک شدند");
   }
   else if(keycode == 72) // کلید H
   {
      ShowHelp();
   }
}

//+------------------------------------------------------------------+
//| مدیریت کشیدن آبجکت‌ها                                           |
//+------------------------------------------------------------------+
void HandleDragEvent(string objname)
{
   double newprice = 0.0;
   
   if(ObjectGetDouble(0, objname, OBJPROP_PRICE, 0, newprice))
   {
      if(StringFind(objname, "Entry") != -1)
      {
         entryPrice = newprice;
         Print("✏️ قیمت ورود به روز شد: ", DoubleToString(entryPrice, _Digits));
      }
      else if(StringFind(objname, "SL") != -1)
      {
         slPrice = newprice;
         Print("✏️ استاپ لاس به روز شد: ", DoubleToString(slPrice, _Digits));
      }
      else if(StringFind(objname, "TP1") != -1)
      {
         tp1Price = newprice;
         Print("✏️ TP1 به روز شد: ", DoubleToString(tp1Price, _Digits));
      }
      else if(StringFind(objname, "TP2") != -1 && InpEnableTP2)
      {
         tp2Price = newprice;
         Print("✏️ TP2 به روز شد: ", DoubleToString(tp2Price, _Digits));
      }
      else if(StringFind(objname, "TP3") != -1 && InpEnableTP3)
      {
         tp3Price = newprice;
         Print("✏️ TP3 به روز شد: ", DoubleToString(tp3Price, _Digits));
      }
      
      isLong = (slPrice < entryPrice);
      RefreshAll();
   }
}

//+------------------------------------------------------------------+
//| محاسبه سطوح Take Profit                                         |
//+------------------------------------------------------------------+
void CalculateTPLevels()
{
   double risk = MathAbs(entryPrice - slPrice);
   
   // محاسبه TP1 بر اساس نسبت ریسک به ریوارد
   tp1Price = isLong ? entryPrice + risk * InpRiskReward : entryPrice - risk * InpRiskReward;
   
   // محاسبه TP2 و TP3 در صورت فعال بودن
   if(InpEnableTP2)
      tp2Price = isLong ? entryPrice + risk * InpRiskReward * InpTP2Multiplier : entryPrice - risk * InpRiskReward * InpTP2Multiplier;
   
   if(InpEnableTP3)
      tp3Price = isLong ? entryPrice + risk * InpRiskReward * InpTP3Multiplier : entryPrice - risk * InpRiskReward * InpTP3Multiplier;
      
   // اعمال ریسک ثابت در صورت تنظیم
   if(InpFixedRiskPips > 0)
   {
      double riskPoints = InpFixedRiskPips * _Point;
      slPrice = isLong ? entryPrice - riskPoints : entryPrice + riskPoints;
   }
}

//+------------------------------------------------------------------+
//| رسم تمام آبجکت‌ها                                               |
//+------------------------------------------------------------------+
void DrawAllObjects()
{
   // رسم خطوط اصلی
   DrawHorizontalLine("Entry", entryPrice, InpEntryColor);
   DrawHorizontalLine("SL", slPrice, InpSLColor);
   DrawHorizontalLine("TP1", tp1Price, InpTP1Color);
   
   if(InpEnableTP2)
      DrawHorizontalLine("TP2", tp2Price, InpTP2Color);
   
   if(InpEnableTP3)
      DrawHorizontalLine("TP3", tp3Price, InpTP3Color);
   
   // رسم نواحی رنگی
   if(InpShowRiskArea)
      DrawRiskArea();
   
   if(InpShowRewardArea)
      DrawRewardArea();
   
   // رسم لیبل‌ها
   if(InpShowLabels)
      DrawLabels();
}

//+------------------------------------------------------------------+
//| رسم خط افقی                                                     |
//+------------------------------------------------------------------+
void DrawHorizontalLine(string name, double price, color clr)
{
   string objname = objPrefix + name;
   
   // حذف آبجکت قبلی
   ObjectDelete(0, objname);
   
   // ایجاد خط جدید
   if(ObjectCreate(0, objname, OBJ_HLINE, 0, 0, price))
   {
      ObjectSetInteger(0, objname, OBJPROP_COLOR, clr);
      ObjectSetInteger(0, objname, OBJPROP_STYLE, STYLE_SOLID);
      ObjectSetInteger(0, objname, OBJPROP_WIDTH, InpLineWidth);
      ObjectSetInteger(0, objname, OBJPROP_SELECTABLE, true);
      ObjectSetInteger(0, objname, OBJPROP_BACK, false);
      
      // اضافه کردن متن
      if(InpShowLabels)
      {
         string text = name + ": " + DoubleToString(price, _Digits);
         ObjectSetString(0, objname, OBJPROP_TEXT, text);
      }
   }
}

//+------------------------------------------------------------------+
//| رسم ناحیه ریسک                                                  |
//+------------------------------------------------------------------+
void DrawRiskArea()
{
   string areaname = objPrefix + "RiskArea";
   ObjectDelete(0, areaname);
   
   datetime startTime = iTime(_Symbol, _Period, 50);
   datetime endTime = iTime(_Symbol, _Period, 0);
   double minPrice = MathMin(entryPrice, slPrice);
   double maxPrice = MathMax(entryPrice, slPrice);
   
   if(ObjectCreate(0, areaname, OBJ_RECTANGLE, 0, startTime, minPrice, endTime, maxPrice))
   {
      ObjectSetInteger(0, areaname, OBJPROP_COLOR, InpRiskColor);
      ObjectSetInteger(0, areaname, OBJPROP_BGCOLOR, InpRiskColor);
      ObjectSetInteger(0, areaname, OBJPROP_FILL, true);
      ObjectSetInteger(0, areaname, OBJPROP_BACK, true);
      ObjectSetInteger(0, areaname, OBJPROP_SELECTABLE, false);
   }
}

//+------------------------------------------------------------------+
//| رسم ناحیه ریوارد                                                |
//+------------------------------------------------------------------+
void DrawRewardArea()
{
   string areaname = objPrefix + "RewardArea";
   ObjectDelete(0, areaname);
   
   datetime startTime = iTime(_Symbol, _Period, 50);
   datetime endTime = iTime(_Symbol, _Period, 0);
   double minPrice = MathMin(entryPrice, tp1Price);
   double maxPrice = MathMax(entryPrice, tp1Price);
   
   if(ObjectCreate(0, areaname, OBJ_RECTANGLE, 0, startTime, minPrice, endTime, maxPrice))
   {
      ObjectSetInteger(0, areaname, OBJPROP_COLOR, InpRewardColor);
      ObjectSetInteger(0, areaname, OBJPROP_BGCOLOR, InpRewardColor);
      ObjectSetInteger(0, areaname, OBJPROP_FILL, true);
      ObjectSetInteger(0, areaname, OBJPROP_BACK, true);
      ObjectSetInteger(0, areaname, OBJPROP_SELECTABLE, false);
   }
}

//+------------------------------------------------------------------+
//| رسم لیبل‌های اطلاعاتی                                           |
//+------------------------------------------------------------------+
void DrawLabels()
{
   DrawPriceLabel("EntryLabel", entryPrice, InpEntryColor, "ENTRY");
   DrawPriceLabel("SLLabel", slPrice, InpSLColor, "STOP LOSS");
   DrawPriceLabel("TP1Label", tp1Price, InpTP1Color, "TP1");
   
   if(InpEnableTP2)
      DrawPriceLabel("TP2Label", tp2Price, InpTP2Color, "TP2");
   
   if(InpEnableTP3)
      DrawPriceLabel("TP3Label", tp3Price, InpTP3Color, "TP3");
}

//+------------------------------------------------------------------+
//| رسم لیبل قیمت                                                   |
//+------------------------------------------------------------------+
void DrawPriceLabel(string name, double price, color clr, string text)
{
   string objname = objPrefix + name;
   ObjectDelete(0, objname);
   
   datetime labelTime = iTime(_Symbol, _Period, 10);
   
   if(ObjectCreate(0, objname, OBJ_TEXT, 0, labelTime, price))
   {
      ObjectSetString(0, objname, OBJPROP_TEXT, text);
      ObjectSetInteger(0, objname, OBJPROP_COLOR, clr);
      ObjectSetInteger(0, objname, OBJPROP_FONTSIZE, 10);
      ObjectSetInteger(0, objname, OBJPROP_BACK, false);
      ObjectSetInteger(0, objname, OBJPROP_SELECTABLE, false);
   }
}

//+------------------------------------------------------------------+
//| بروزرسانی پنل اطلاعات                                           |
//+------------------------------------------------------------------+
void UpdateInfoPanel()
{
   if(!InpShowInfo) return;
   
   double riskPips = MathAbs(entryPrice - slPrice) / _Point;
   double tp1Pips = MathAbs(tp1Price - entryPrice) / _Point;
   double actualRR = tp1Pips / riskPips;
   
   // محاسبه ارزش دلاری
   double tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   double pipValue = (tickSize > 0) ? (tickValue / tickSize * _Point) : 0;
   double riskDollar = riskPips * pipValue * InpLotSize;
   double rewardDollar = tp1Pips * pipValue * InpLotSize;
   
   string info = "";
   info += "🎯 RiskToReward Pro\n";
   info += "═══════════════════\n";
   info += "📍 ورود: " + DoubleToString(entryPrice, _Digits) + "\n";
   info += "🛑 استاپ: " + DoubleToString(slPrice, _Digits);
   info += " (" + DoubleToString(riskPips, 1) + " پیپ";
   info += " | $" + DoubleToString(riskDollar, 2) + ")\n";
   
   info += "🎯 TP1: " + DoubleToString(tp1Price, _Digits);
   info += " (" + DoubleToString(tp1Pips, 1) + " پیپ";
   info += " | $" + DoubleToString(rewardDollar, 2) + ")\n";
   
   info += "📊 نسبت: 1:" + DoubleToString(actualRR, 2) + "\n";
   
   if(InpEnableTP2)
   {
      double tp2Pips = MathAbs(tp2Price - entryPrice) / _Point;
      info += "🎯 TP2: " + DoubleToString(tp2Price, _Digits);
      info += " (" + DoubleToString(tp2Pips, 1) + " پیپ)\n";
   }
   
   if(InpEnableTP3)
   {
      double tp3Pips = MathAbs(tp3Price - entryPrice) / _Point;
      info += "🎯 TP3: " + DoubleToString(tp3Price, _Digits);
      info += " (" + DoubleToString(tp3Pips, 1) + " پیپ)\n";
   }
   
   info += "═══════════════════\n";
   info += "💡 راهنما:\n";
   info += "• دو بار کلیک = تنظیم سطوح\n";
   info += "• R = ریست اندیکاتور\n";
   info += "• C = پاک کردن آبجکت‌ها\n";
   info += "• H = نمایش کمک";
   
   Comment(info);
}

//+------------------------------------------------------------------+
//| بررسی هشدارها                                                   |
//+------------------------------------------------------------------+
void CheckAlerts()
{
   if(TimeCurrent() < lastAlertTime + 5) // جلوگیری از هشدارهای مکرر
      return;
   
   double currentPrice = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   string alertMessage = "";
   
   // بررسی برخورد به استاپ لاس
   if((isLong && currentPrice <= slPrice) || (!isLong && currentPrice >= slPrice))
   {
      alertMessage = "🚨 استاپ لاس فعال شد! قیمت: " + DoubleToString(currentPrice, _Digits);
   }
   // بررسی برخورد به TP1
   else if((isLong && currentPrice >= tp1Price) || (!isLong && currentPrice <= tp1Price))
   {
      alertMessage = "🎯 TP1 فعال شد! قیمت: " + DoubleToString(currentPrice, _Digits);
   }
   // بررسی برخورد به TP2
   else if(InpEnableTP2 && ((isLong && currentPrice >= tp2Price) || (!isLong && currentPrice <= tp2Price)))
   {
      alertMessage = "🎯 TP2 فعال شد! قیمت: " + DoubleToString(currentPrice, _Digits);
   }
   // بررسی برخورد به TP3
   else if(InpEnableTP3 && ((isLong && currentPrice >= tp3Price) || (!isLong && currentPrice <= tp3Price)))
   {
      alertMessage = "🎯 TP3 فعال شد! قیمت: " + DoubleToString(currentPrice, _Digits);
   }
   
   if(alertMessage != "")
   {
      Alert(alertMessage);
      Print(alertMessage);
      
      if(InpPlaySound)
         PlaySound(InpSoundFile);
         
      lastAlertTime = TimeCurrent();
   }
}

//+------------------------------------------------------------------+
//| نمایش راهنما                                                    |
//+------------------------------------------------------------------+
void ShowHelp()
{
   string help = "";
   help += "🎯 راهنمای RiskToReward Pro\n";
   help += "══════════════════════════\n";
   help += "📌 روش کار:\n";
   help += "• دو بار کلیک روی چارت برای تنظیم SL و Entry\n";
   help += "• اولین کلیک = استاپ لاس\n";
   help += "• دومین کلیک = نقطه ورود\n";
   help += "• خطوط به صورت خودکار رسم می‌شوند\n\n";
   help += "⌨️ کلیدهای میانبر:\n";
   help += "• R = ریست اندیکاتور\n";
   help += "• C = پاک کردن تمام خطوط\n";
   help += "• H = نمایش این راهنما\n\n";
   help += "⚙️ تنظیمات مهم:\n";
   help += "• Risk/Reward = نسبت ریسک به ریوارد\n";
   help += "• Fixed Risk Pips = ریسک ثابت بر اساس پیپ\n";
   help += "• Enable TP2/TP3 = سطوح سود اضافی\n";
   help += "══════════════════════════";
   
   Comment(help);
}

//+------------------------------------------------------------------+
//| پاک کردن تمام آبجکت‌ها                                          |
//+------------------------------------------------------------------+
void DeleteAllObjects()
{
   int total = ObjectsTotal(0);
   for(int i = total - 1; i >= 0; i--)
   {
      string name = ObjectName(0, i);
      if(StringFind(name, objPrefix) == 0)
         ObjectDelete(0, name);
   }
}

//+------------------------------------------------------------------+
//| ریست اندیکاتور                                                  |
//+------------------------------------------------------------------+
void ResetIndicator()
{
   DeleteAllObjects();
   clickCount = 0;
   
   // تنظیم مجدد مقادیر اولیه
   entryPrice = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   slPrice = entryPrice - 100 * _Point;
   isLong = true;
   
   RefreshAll();
}

//+------------------------------------------------------------------+
//| بروزرسانی کامل                                                  |
//+------------------------------------------------------------------+
void RefreshAll()
{
   CalculateTPLevels();
   DeleteAllObjects();
   DrawAllObjects();
   UpdateInfoPanel();
}

//+------------------------------------------------------------------+
//| تابع خاتمه                                                      |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   DeleteAllObjects();
   Comment("");
   Print("=== RiskToReward Pro Indicator Removed ===");
}
//+------------------------------------------------------------------+

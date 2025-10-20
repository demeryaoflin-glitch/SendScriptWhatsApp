# SendScriptWhatsAppimport React, { useState, useEffect, useRef } from "react";
import io from "socket.io-client";
import axios from "axios";
import { 
  LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, 
  ResponsiveContainer, BarChart, Bar 
} from 'recharts';

const SOCKET_URL = "http://localhost:8000";
const socket = io(SOCKET_URL);

export default function App() {
  const [telemetry, setTelemetry] = useState(null);
  const [commands, setCommands] = useState([]);
  const [cmdType, setCmdType] = useState("ATTITUDE_ADJUST");
  const [cmdParams, setCmdParams] = useState({});
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [telemetryHistory, setTelemetryHistory] = useState([]);
  const historyRef = useRef([]);

  useEffect(() => {
    const handleTelemetry = (msg) => {
      const newTelemetry = msg.data;
      setTelemetry(newTelemetry);
      
      // حفظ التاريخ للرسوم البيانية
      historyRef.current = [...historyRef.current.slice(-50), {
        timestamp: new Date().toLocaleTimeString(),
        battery: newTelemetry.power.battery,
        temperature: newTelemetry.temperature,
        roll: newTelemetry.attitude.roll,
        pitch: newTelemetry.attitude.pitch,
        yaw: newTelemetry.attitude.yaw
      }];
      setTelemetryHistory(historyRef.current);
    };

    const handleCommandUpdate = () => {
      fetchCommands();
    };

    socket.on("telemetry", handleTelemetry);
    socket.on("command_update", handleCommandUpdate);
    
    fetchInitialData();

    return () => {
      socket.off("telemetry", handleTelemetry);
      socket.off("command_update", handleCommandUpdate);
    };
  }, []);

  const fetchInitialData = async () => {
    try {
      setLoading(true);
      const [commandsRes, telemetryRes] = await Promise.all([
        axios.get(`${SOCKET_URL}/api/commands`),
        axios.get(`${SOCKET_URL}/api/telemetry/latest`)
      ]);
      setCommands(commandsRes.data);
      setTelemetry(telemetryRes.data.data);
    } catch (err) {
      setError("فشل في الاتصال بالخادم");
      console.error("Error:", err);
    } finally {
      setLoading(false);
    }
  };

  const fetchCommands = async () => {
    try {
      const response = await axios.get(`${SOCKET_URL}/api/commands`);
      setCommands(response.data);
    } catch (err) {
      setError("فشل في جلب الأوامر");
    }
  };

  const sendCommand = async () => {
    try {
      setLoading(true);
      setError(null);
      
      const payload = { 
        type: cmdType, 
        params: {
          ...cmdParams,
          note: "من واجهة التحكم",
          timestamp: new Date().toISOString()
        } 
      };
      
      await axios.post(`${SOCKET_URL}/api/commands`, payload);
      
      // إعادة تعيين المعاملات بعد الإرسال
      setCmdParams({});
    } catch (err) {
      setError("فشل في إرسال الأمر");
      console.error("Command error:", err);
    } finally {
      setLoading(false);
    }
  };

  const renderCommandParams = () => {
    switch (cmdType) {
      case 'ATTITUDE_ADJUST':
        return (
          <div className="grid grid-cols-3 gap-2 mt-2">
            <input
              type="number"
              placeholder="Roll"
              value={cmdParams.roll || ''}
              onChange={(e) => setCmdParams({...cmdParams, roll: parseFloat(e.target.value)})}
              className="border p-1 rounded text-black"
            />
            <input
              type="number"
              placeholder="Pitch"
              value={cmdParams.pitch || ''}
              onChange={(e) => setCmdParams({...cmdParams, pitch: parseFloat(e.target.value)})}
              className="border p-1 rounded text-black"
            />
            <input
              type="number"
              placeholder="Yaw"
              value={cmdParams.yaw || ''}
              onChange={(e) => setCmdParams({...cmdParams, yaw: parseFloat(e.target.value)})}
              className="border p-1 rounded text-black"
            />
          </div>
        );
      
      case 'PAYLOAD_ON':
      case 'PAYLOAD_OFF':
        return (
          <select
            value={cmdParams.device || ''}
            onChange={(e) => setCmdParams({...cmdParams, device: e.target.value})}
            className="border p-1 rounded text-black mt-2 w-full"
          >
            <option value="">اختر الجهاز</option>
            <option value="camera">الكاميرا</option>
            <option value="comms">الاتصالات</option>
          </select>
        );
      
      default:
        return null;
    }
  };

  if (loading && !telemetry) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-gray-900 to-blue-900 flex items-center justify-center">
        <div className="text-white text-xl">جاري التحميل...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-gray-900 to-blue-900 p-6">
      <div className="max-w-7xl mx-auto">
        {/* Header */}
        <header className="text-center mb-8">
          <h1 className="text-4xl font-bold text-white mb-2">
            نظام التحكم الأرضي للأقمار الصناعية
          </h1>
          <p className="text-blue-200">محطة مراقبة وتحكم في الوقت الحقيقي</p>
        </header>

        {error && (
          <div className="bg-red-500 text-white p-4 rounded-lg mb-6 text-center">
            {error}
          </div>
        )}

        <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
          {/* لوحة التليمتري */}
          <div className="lg:col-span-2 space-y-6">
            {/* حالة القمر */}
            <div className="bg-gray-800 rounded-xl p-6 shadow-2xl">
              <h2 className="text-2xl font-bold text-white mb-4 border-b border-blue-500 pb-2">
                📊 حالة القمر الصناعي
              </h2>
              
              {telemetry && (
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  {/* الاتجاه */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-blue-300 mb-2">الاتجاه</h3>
                    <div className="space-y-1 text-sm">
                      <div className="flex justify-between">
                        <span>Roll:</span>
                        <span className="font-mono">{telemetry.attitude.roll.toFixed(2)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>Pitch:</span>
                        <span className="font-mono">{telemetry.attitude.pitch.toFixed(2)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>Yaw:</span>
                        <span className="font-mono">{telemetry.attitude.yaw.toFixed(2)}°</span>
                      </div>
                    </div>
                  </div>

                  {/* الطاقة */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-green-300 mb-2">الطاقة</h3>
                    <div className="space-y-2">
                      <div>
                        <div className="flex justify-between text-sm mb-1">
                          <span>البطارية:</span>
                          <span className="font-mono">{telemetry.power.battery.toFixed(1)}%</span>
                        </div>
                        <div className="w-full bg-gray-600 rounded-full h-2">
                          <div 
                            className="bg-green-500 h-2 rounded-full transition-all"
                            style={{ width: `${telemetry.power.battery}%` }}
                          ></div>
                        </div>
                      </div>
                      <div>
                        <div className="flex justify-between text-sm mb-1">
                          <span>الخلايا الشمسية:</span>
                          <span className="font-mono">{telemetry.power.solar.toFixed(1)}%</span>
                        </div>
                        <div className="w-full bg-gray-600 rounded-full h-2">
                          <div 
                            className="bg-yellow-500 h-2 rounded-full transition-all"
                            style={{ width: `${telemetry.power.solar}%` }}
                          ></div>
                        </div>
                      </div>
                    </div>
                  </div>

                  {/* الموقع */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-purple-300 mb-2">الموقع</h3>
                    <div className="space-y-1 text-sm">
                      <div className="flex justify-between">
                        <span>خط العرض:</span>
                        <span className="font-mono">{telemetry.position.latitude.toFixed(4)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>خط الطول:</span>
                        <span className="font-mono">{telemetry.position.longitude.toFixed(4)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>الارتفاع:</span>
                        <span className="font-mono">{telemetry.position.altitude.toFixed(0)} km</span>
                      </div>
                    </div>
                  </div>

                  {/* الحمولة */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-red-300 mb-2">الحمولة</h3>
                    <div className="space-y-2 text-sm">
                      <div className="flex justify-between">
                        <span>الكاميرا:</span>
                        <span className={`font-mono ${
                          telemetry.payload.camera === 'ON' ? 'text-green-400' : 'text-red-400'
                        }`}>
                          {telemetry.payload.camera}
                        </span>
                      </div>
                      <div className="flex justify-between">
                        <span>الاتصالات:</span>
                        <span className={`font-mono ${
                          telemetry.payload.comms === 'ON' ? 'text-green-400' : 'text-red-400'
                        }`}>
                          {telemetry.payload.comms}
                        </span>
                      </div>
                      <div className="flex justify-between">import React, { useState, useEffect, useRef } from "react";
import io from "socket.io-client";
import axios from "axios";
import { 
  LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, 
  ResponsiveContainer, BarChart, Bar 
} from 'recharts';

const SOCKET_URL = "http://localhost:8000";
const socket = io(SOCKET_URL);

export default function App() {
  const [telemetry, setTelemetry] = useState(null);
  const [commands, setCommands] = useState([]);
  const [cmdType, setCmdType] = useState("ATTITUDE_ADJUST");
  const [cmdParams, setCmdParams] = useState({});
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [telemetryHistory, setTelemetryHistory] = useState([]);
  const historyRef = useRef([]);

  useEffect(() => {
    const handleTelemetry = (msg) => {
      const newTelemetry = msg.data;
      setTelemetry(newTelemetry);
      
      // حفظ التاريخ للرسوم البيانية
      historyRef.current = [...historyRef.current.slice(-50), {
        timestamp: new Date().toLocaleTimeString(),
        battery: newTelemetry.power.battery,
        temperature: newTelemetry.temperature,
        roll: newTelemetry.attitude.roll,
        pitch: newTelemetry.attitude.pitch,
        yaw: newTelemetry.attitude.yaw
      }];
      setTelemetryHistory(historyRef.current);
    };

    const handleCommandUpdate = () => {
      fetchCommands();
    };

    socket.on("telemetry", handleTelemetry);
    socket.on("command_update", handleCommandUpdate);
    
    fetchInitialData();

    return () => {
      socket.off("telemetry", handleTelemetry);
      socket.off("command_update", handleCommandUpdate);
    };
  }, []);

  const fetchInitialData = async () => {
    try {
      setLoading(true);
      const [commandsRes, telemetryRes] = await Promise.all([
        axios.get(`${SOCKET_URL}/api/commands`),
        axios.get(`${SOCKET_URL}/api/telemetry/latest`)
      ]);
      setCommands(commandsRes.data);
      setTelemetry(telemetryRes.data.data);
    } catch (err) {
      setError("فشل في الاتصال بالخادم");
      console.error("Error:", err);
    } finally {
      setLoading(false);
    }
  };

  const fetchCommands = async () => {
    try {
      const response = await axios.get(`${SOCKET_URL}/api/commands`);
      setCommands(response.data);
    } catch (err) {
      setError("فشل في جلب الأوامر");
    }
  };

  const sendCommand = async () => {
    try {
      setLoading(true);
      setError(null);
      
      const payload = { 
        type: cmdType, 
        params: {
          ...cmdParams,
          note: "من واجهة التحكم",
          timestamp: new Date().toISOString()
        } 
      };
      
      await axios.post(`${SOCKET_URL}/api/commands`, payload);
      
      // إعادة تعيين المعاملات بعد الإرسال
      setCmdParams({});
    } catch (err) {
      setError("فشل في إرسال الأمر");
      console.error("Command error:", err);
    } finally {
      setLoading(false);
    }
  };

  const renderCommandParams = () => {
    switch (cmdType) {
      case 'ATTITUDE_ADJUST':
        return (
          <div className="grid grid-cols-3 gap-2 mt-2">
            <input
              type="number"
              placeholder="Roll"
              value={cmdParams.roll || ''}
              onChange={(e) => setCmdParams({...cmdParams, roll: parseFloat(e.target.value)})}
              className="border p-1 rounded text-black"
            />
            <input
              type="number"
              placeholder="Pitch"
              value={cmdParams.pitch || ''}
              onChange={(e) => setCmdParams({...cmdParams, pitch: parseFloat(e.target.value)})}
              className="border p-1 rounded text-black"
            />
            <input
              type="number"
              placeholder="Yaw"
              value={cmdParams.yaw || ''}
              onChange={(e) => setCmdParams({...cmdParams, yaw: parseFloat(e.target.value)})}
              className="border p-1 rounded text-black"
            />
          </div>
        );
      
      case 'PAYLOAD_ON':
      case 'PAYLOAD_OFF':
        return (
          <select
            value={cmdParams.device || ''}
            onChange={(e) => setCmdParams({...cmdParams, device: e.target.value})}
            className="border p-1 rounded text-black mt-2 w-full"
          >
            <option value="">اختر الجهاز</option>
            <option value="camera">الكاميرا</option>
            <option value="comms">الاتصالات</option>
          </select>
        );
      
      default:
        return null;
    }
  };

  if (loading && !telemetry) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-gray-900 to-blue-900 flex items-center justify-center">
        <div className="text-white text-xl">جاري التحميل...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-gray-900 to-blue-900 p-6">
      <div className="max-w-7xl mx-auto">
        {/* Header */}
        <header className="text-center mb-8">
          <h1 className="text-4xl font-bold text-white mb-2">
            نظام التحكم الأرضي للأقمار الصناعية
          </h1>
          <p className="text-blue-200">محطة مراقبة وتحكم في الوقت الحقيقي</p>
        </header>

        {error && (
          <div className="bg-red-500 text-white p-4 rounded-lg mb-6 text-center">
            {error}
          </div>
        )}

        <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
          {/* لوحة التليمتري */}
          <div className="lg:col-span-2 space-y-6">
            {/* حالة القمر */}
            <div className="bg-gray-800 rounded-xl p-6 shadow-2xl">
              <h2 className="text-2xl font-bold text-white mb-4 border-b border-blue-500 pb-2">
                📊 حالة القمر الصناعي
              </h2>
              
              {telemetry && (
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  {/* الاتجاه */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-blue-300 mb-2">الاتجاه</h3>
                    <div className="space-y-1 text-sm">
                      <div className="flex justify-between">
                        <span>Roll:</span>
                        <span className="font-mono">{telemetry.attitude.roll.toFixed(2)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>Pitch:</span>
                        <span className="font-mono">{telemetry.attitude.pitch.toFixed(2)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>Yaw:</span>
                        <span className="font-mono">{telemetry.attitude.yaw.toFixed(2)}°</span>
                      </div>
                    </div>
                  </div>

                  {/* الطاقة */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-green-300 mb-2">الطاقة</h3>
                    <div className="space-y-2">
                      <div>
                        <div className="flex justify-between text-sm mb-1">
                          <span>البطارية:</span>
                          <span className="font-mono">{telemetry.power.battery.toFixed(1)}%</span>
                        </div>
                        <div className="w-full bg-gray-600 rounded-full h-2">
                          <div 
                            className="bg-green-500 h-2 rounded-full transition-all"
                            style={{ width: `${telemetry.power.battery}%` }}
                          ></div>
                        </div>
                      </div>
                      <div>
                        <div className="flex justify-between text-sm mb-1">
                          <span>الخلايا الشمسية:</span>
                          <span className="font-mono">{telemetry.power.solar.toFixed(1)}%</span>
                        </div>
                        <div className="w-full bg-gray-600 rounded-full h-2">
                          <div 
                            className="bg-yellow-500 h-2 rounded-full transition-all"
                            style={{ width: `${telemetry.power.solar}%` }}
                          ></div>
                        </div>
                      </div>
                    </div>
                  </div>

                  {/* الموقع */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-purple-300 mb-2">الموقع</h3>
                    <div className="space-y-1 text-sm">
                      <div className="flex justify-between">
                        <span>خط العرض:</span>
                        <span className="font-mono">{telemetry.position.latitude.toFixed(4)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>خط الطول:</span>
                        <span className="font-mono">{telemetry.position.longitude.toFixed(4)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>الارتفاع:</span>
                        <span className="font-mono">{telemetry.position.altitude.toFixed(0)} km</span>
                      </div>
                    </div>
                  </div>

                  {/* الحمولة */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-red-300 mb-2">الحمولة</h3>
                    <div className="space-y-2 text-sm">
                      <div className="flex justify-between">
                        <span>الكاميرا:</span>
                        <span className={`font-mono ${
                          telemetry.payload.camera === 'ON' ? 'text-green-400' : 'text-red-400'
                        }`}>
                          {telemetry.payload.camera}
                        </span>
                      </div>
                      <div className="flex justify-between">
                        <span>الاتصالات:</span>
                        <span className={`font-mono ${
                          telemetry.payload.comms === 'ON' ? 'text-green-400' : 'text-red-400'
                        }`}>
                          {telemetry.payload.comms}
                        </span>
                      </div>
                      <div className="flex justify-between">import React, { useState, useEffect, useRef } from "react";
import io from "socket.io-client";
import axios from "axios";
import { 
  LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, 
  ResponsiveContainer, BarChart, Bar 
} from 'recharts';

const SOCKET_URL = "http://localhost:8000";
const socket = io(SOCKET_URL);

export default function App() {
  const [telemetry, setTelemetry] = useState(null);
  const [commands, setCommands] = useState([]);
  const [cmdType, setCmdType] = useState("ATTITUDE_ADJUST");
  const [cmdParams, setCmdParams] = useState({});
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [telemetryHistory, setTelemetryHistory] = useState([]);
  const historyRef = useRef([]);

  useEffect(() => {
    const handleTelemetry = (msg) => {
      const newTelemetry = msg.data;
      setTelemetry(newTelemetry);
      
      // حفظ التاريخ للرسوم البيانية
      historyRef.current = [...historyRef.current.slice(-50), {
        timestamp: new Date().toLocaleTimeString(),
        battery: newTelemetry.power.battery,
        temperature: newTelemetry.temperature,
        roll: newTelemetry.attitude.roll,
        pitch: newTelemetry.attitude.pitch,
        yaw: newTelemetry.attitude.yaw
      }];
      setTelemetryHistory(historyRef.current);
    };

    const handleCommandUpdate = () => {
      fetchCommands();
    };

    socket.on("telemetry", handleTelemetry);
    socket.on("command_update", handleCommandUpdate);
    
    fetchInitialData();

    return () => {
      socket.off("telemetry", handleTelemetry);
      socket.off("command_update", handleCommandUpdate);
    };
  }, []);

  const fetchInitialData = async () => {
    try {
      setLoading(true);
      const [commandsRes, telemetryRes] = await Promise.all([
        axios.get(`${SOCKET_URL}/api/commands`),
        axios.get(`${SOCKET_URL}/api/telemetry/latest`)
      ]);
      setCommands(commandsRes.data);
      setTelemetry(telemetryRes.data.data);
    } catch (err) {
      setError("فشل في الاتصال بالخادم");
      console.error("Error:", err);
    } finally {
      setLoading(false);
    }
  };

  const fetchCommands = async () => {
    try {
      const response = await axios.get(`${SOCKET_URL}/api/commands`);
      setCommands(response.data);
    } catch (err) {
      setError("فشل في جلب الأوامر");
    }
  };

  const sendCommand = async () => {
    try {
      setLoading(true);
      setError(null);
      
      const payload = { 
        type: cmdType, 
        params: {
          ...cmdParams,
          note: "من واجهة التحكم",
          timestamp: new Date().toISOString()
        } 
      };
      
      await axios.post(`${SOCKET_URL}/api/commands`, payload);
      
      // إعادة تعيين المعاملات بعد الإرسال
      setCmdParams({});
    } catch (err) {
      setError("فشل في إرسال الأمر");
      console.error("Command error:", err);
    } finally {
      setLoading(false);
    }
  };

  const renderCommandParams = () => {
    switch (cmdType) {
      case 'ATTITUDE_ADJUST':
        return (
          <div className="grid grid-cols-3 gap-2 mt-2">
            <input
              type="number"
              placeholder="Roll"
              value={cmdParams.roll || ''}
              onChange={(e) => setCmdParams({...cmdParams, roll: parseFloat(e.target.value)})}
              className="border p-1 rounded text-black"
            />
            <input
              type="number"
              placeholder="Pitch"
              value={cmdParams.pitch || ''}
              onChange={(e) => setCmdParams({...cmdParams, pitch: parseFloat(e.target.value)})}
              className="border p-1 rounded text-black"
            />
            <input
              type="number"
              placeholder="Yaw"
              value={cmdParams.yaw || ''}
              onChange={(e) => setCmdParams({...cmdParams, yaw: parseFloat(e.target.value)})}
              className="border p-1 rounded text-black"
            />
          </div>
        );
      
      case 'PAYLOAD_ON':
      case 'PAYLOAD_OFF':
        return (
          <select
            value={cmdParams.device || ''}
            onChange={(e) => setCmdParams({...cmdParams, device: e.target.value})}
            className="border p-1 rounded text-black mt-2 w-full"
          >
            <option value="">اختر الجهاز</option>
            <option value="camera">الكاميرا</option>
            <option value="comms">الاتصالات</option>
          </select>
        );
      
      default:
        return null;
    }
  };

  if (loading && !telemetry) {
    return (
      <div className="min-h-screen bg-gradient-to-br from-gray-900 to-blue-900 flex items-center justify-center">
        <div className="text-white text-xl">جاري التحميل...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-gray-900 to-blue-900 p-6">
      <div className="max-w-7xl mx-auto">
        {/* Header */}
        <header className="text-center mb-8">
          <h1 className="text-4xl font-bold text-white mb-2">
            نظام التحكم الأرضي للأقمار الصناعية
          </h1>
          <p className="text-blue-200">محطة مراقبة وتحكم في الوقت الحقيقي</p>
        </header>

        {error && (
          <div className="bg-red-500 text-white p-4 rounded-lg mb-6 text-center">
            {error}
          </div>
        )}

        <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
          {/* لوحة التليمتري */}
          <div className="lg:col-span-2 space-y-6">
            {/* حالة القمر */}
            <div className="bg-gray-800 rounded-xl p-6 shadow-2xl">
              <h2 className="text-2xl font-bold text-white mb-4 border-b border-blue-500 pb-2">
                📊 حالة القمر الصناعي
              </h2>
              
              {telemetry && (
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  {/* الاتجاه */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-blue-300 mb-2">الاتجاه</h3>
                    <div className="space-y-1 text-sm">
                      <div className="flex justify-between">
                        <span>Roll:</span>
                        <span className="font-mono">{telemetry.attitude.roll.toFixed(2)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>Pitch:</span>
                        <span className="font-mono">{telemetry.attitude.pitch.toFixed(2)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>Yaw:</span>
                        <span className="font-mono">{telemetry.attitude.yaw.toFixed(2)}°</span>
                      </div>
                    </div>
                  </div>

                  {/* الطاقة */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-green-300 mb-2">الطاقة</h3>
                    <div className="space-y-2">
                      <div>
                        <div className="flex justify-between text-sm mb-1">
                          <span>البطارية:</span>
                          <span className="font-mono">{telemetry.power.battery.toFixed(1)}%</span>
                        </div>
                        <div className="w-full bg-gray-600 rounded-full h-2">
                          <div 
                            className="bg-green-500 h-2 rounded-full transition-all"
                            style={{ width: `${telemetry.power.battery}%` }}
                          ></div>
                        </div>
                      </div>
                      <div>
                        <div className="flex justify-between text-sm mb-1">
                          <span>الخلايا الشمسية:</span>
                          <span className="font-mono">{telemetry.power.solar.toFixed(1)}%</span>
                        </div>
                        <div className="w-full bg-gray-600 rounded-full h-2">
                          <div 
                            className="bg-yellow-500 h-2 rounded-full transition-all"
                            style={{ width: `${telemetry.power.solar}%` }}
                          ></div>
                        </div>
                      </div>
                    </div>
                  </div>

                  {/* الموقع */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-purple-300 mb-2">الموقع</h3>
                    <div className="space-y-1 text-sm">
                      <div className="flex justify-between">
                        <span>خط العرض:</span>
                        <span className="font-mono">{telemetry.position.latitude.toFixed(4)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>خط الطول:</span>
                        <span className="font-mono">{telemetry.position.longitude.toFixed(4)}°</span>
                      </div>
                      <div className="flex justify-between">
                        <span>الارتفاع:</span>
                        <span className="font-mono">{telemetry.position.altitude.toFixed(0)} km</span>
                      </div>
                    </div>
                  </div>

                  {/* الحمولة */}
                  <div className="bg-gray-700 p-4 rounded-lg">
                    <h3 className="text-lg font-semibold text-red-300 mb-2">الحمولة</h3>
                    <div className="space-y-2 text-sm">
                      <div className="flex justify-between">
                        <span>الكاميرا:</span>
                        <span className={`font-mono ${
                          telemetry.payload.camera === 'ON' ? 'text-green-400' : 'text-red-400'
                        }`}>
                          {telemetry.payload.camera}
                        </span>
                      </div>
                      <div className="flex justify-between">
                        <span>الاتصالات:</span>
                        <span className={`font-mono ${
                          telemetry.payload.comms === 'ON' ? 'text-green-400' : 'text-red-400'
                        }`}>
                          {telemetry.payload.comms}
                        </span>
                      </div>
                      <div className="flex justify-between">

Código para enviar o Script inteiro de Shrek ou Bee Movie para seus amigos ou grupos do WhatsApp

## Utilização

Abra [shrekSendScript.js](https://github.com/Matt-Fontes/SendScriptWhatsApp/blob/main/shrekSendScript.js)
Ou
Abra [beeMovieSendScript.js](https://github.com/Matt-Fontes/SendScriptWhatsApp/blob/main/beeMovieSendScript.js)

Copie todo o conteúdo (clique em raw -> ctrl+a -> ctrl+c)

No WhatsApp Web abra o console do Browser

|  ⚠️ Aviso importante, numa atualização recente do Google Chrome, está sendo impedido que qualquer script seja colado no Console.|
|--|
|  ***Para contornar esse problema, o console do desenvolvedor espera receber um confirmação textual escrevendo no console: "allow pasting"***| 
|Após isso será permitido colar e continuar a execução do script|


Cole o código no console e aperte Enter

Pronto

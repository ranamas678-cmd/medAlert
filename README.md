# medAlert
an app that help people with medical conditions in schools by alerting their teachers by using this app
import { useState, useEffect } from "react";
import { useNavigate } from "react-router";
import { Button } from "./ui/button";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "./ui/card";
import { Badge } from "./ui/badge";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "./ui/tabs";
import { AlertTriangle, MapPin, Clock, LogOut, UserCircle, Phone, CheckCircle } from "lucide-react";
import { mockTeachers, mockStudents } from "../data/mockData";
import { Alert as AlertType } from "../types";
import { toast } from "sonner";
import { LanguageSwitcher } from "./LanguageSwitcher";
import { useLanguage } from "../i18n/LanguageContext";

export function TeacherDashboard() {
  const navigate = useNavigate();
  const { t } = useLanguage();
  const [teacher, setTeacher] = useState<any>(null);
  const [alerts, setAlerts] = useState<AlertType[]>([]);
  const [myStudents, setMyStudents] = useState<any[]>([]);

  useEffect(() => {
    const userId = localStorage.getItem("currentUserId");
    const userType = localStorage.getItem("userType");
    
    if (!userId || userType !== "teacher") {
      navigate("/");
      return;
    }

    const teacherData = mockTeachers.find(t => t.id === userId);
    if (teacherData) {
      setTeacher(teacherData);
      
      // Get students assigned to this teacher
      const students = mockStudents.filter(s => s.teacherIds.includes(userId));
      setMyStudents(students);
    }

    // Load alerts
    loadAlerts();

    // Listen for new alerts
    const handleStorageChange = () => {
      loadAlerts();
    };

    window.addEventListener("storage", handleStorageChange);
    return () => window.removeEventListener("storage", handleStorageChange);
  }, [navigate]);

  const loadAlerts = () => {
    const userId = localStorage.getItem("currentUserId");
    const storedAlerts = JSON.parse(localStorage.getItem("alerts") || "[]");
    
    // Filter alerts for students assigned to this teacher
    const relevantAlerts = storedAlerts.filter((alert: AlertType) => {
      const student = mockStudents.find(s => s.id === alert.studentId);
      return student && student.teacherIds.includes(userId || "");
    });

    setAlerts(relevantAlerts.reverse()); // Most recent first

    // Show toast for active alerts
    const activeAlerts = relevantAlerts.filter((a: AlertType) => a.status === 'active');
    if (activeAlerts.length > 0) {
      toast.error(t('newEmergencyAlert'), {
        description: `${activeAlerts[0].studentName} ${t('needsAssistance')}`,
      });
    }
  };

  const handleRespondToAlert = (alertId: string) => {
    const existingAlerts = JSON.parse(localStorage.getItem("alerts") || "[]");
    const updatedAlerts = existingAlerts.map((alert: AlertType) => {
      if (alert.id === alertId) {
        return { ...alert, status: 'responded' };
      }
      return alert;
    });
    localStorage.setItem("alerts", JSON.stringify(updatedAlerts));
    loadAlerts();
    
    toast.success(t('responseRecorded'), {
      description: t('studentNotified'),
    });
  };

  const handleResolveAlert = (alertId: string) => {
    const existingAlerts = JSON.parse(localStorage.getItem("alerts") || "[]");
    const updatedAlerts = existingAlerts.map((alert: AlertType) => {
      if (alert.id === alertId) {
        return { ...alert, status: 'resolved' };
      }
      return alert;
    });
    localStorage.setItem("alerts", JSON.stringify(updatedAlerts));
    loadAlerts();
    
    toast.info(t('alertResolved'), {
      description: t('alertResolvedDescription'),
    });
  };

  const handleLogout = () => {
    localStorage.removeItem("currentUserId");
    localStorage.removeItem("userType");
    navigate("/");
  };

  const formatTime = (date: Date) => {
    const d = new Date(date);
    return d.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  };

  const getStatusBadge = (status: string) => {
    switch (status) {
      case 'active':
        return <Badge variant="destructive">{t('active')}</Badge>;
      case 'responded':
        return <Badge className="bg-orange-500">{t('responding')}</Badge>;
      case 'resolved':
        return <Badge className="bg-green-500">{t('resolved')}</Badge>;
      default:
        return <Badge variant="secondary">{status}</Badge>;
    }
  };

  const activeAlerts = alerts.filter(a => a.status === 'active');
  const respondedAlerts = alerts.filter(a => a.status === 'responded');

  if (!teacher) return null;

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4">
      <div className="max-w-6xl mx-auto space-y-4">
        {/* Header */}
        <div className="flex items-center justify-between">
          <div>
            <h1 className="text-2xl font-bold text-gray-900">{teacher.name}</h1>
            <p className="text-sm text-gray-600">{teacher.subject} {t('teacher')}</p>
          </div>
          <div className="flex gap-2">
            <LanguageSwitcher />
            <Button variant="outline" size="sm" onClick={handleLogout}>
              <LogOut className="w-4 h-4 mr-2" />
              {t('logout')}
            </Button>
          </div>
        </div>

        {/* Alert Stats */}
        <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
          <Card>
            <CardHeader className="pb-3">
              <CardTitle className="text-sm font-medium text-muted-foreground">
                {t('activeAlerts')}
              </CardTitle>
            </CardHeader>
            <CardContent>
              <p className="text-3xl font-bold text-red-600">{activeAlerts.length}</p>
            </CardContent>
          </Card>
          <Card>
            <CardHeader className="pb-3">
              <CardTitle className="text-sm font-medium text-muted-foreground">
                {t('responding')}
              </CardTitle>
            </CardHeader>
            <CardContent>
              <p className="text-3xl font-bold text-orange-600">{respondedAlerts.length}</p>
            </CardContent>
          </Card>
          <Card>
            <CardHeader className="pb-3">
              <CardTitle className="text-sm font-medium text-muted-foreground">
                {t('myStudents')}
              </CardTitle>
            </CardHeader>
            <CardContent>
              <p className="text-3xl font-bold text-blue-600">{myStudents.length}</p>
            </CardContent>
          </Card>
        </div>

        {/* Main Content */}
        <Tabs defaultValue="alerts" className="space-y-4">
          <TabsList className="grid w-full grid-cols-2">
            <TabsTrigger value="alerts">
              <AlertTriangle className="w-4 h-4 mr-2" />
              {t('emergencyAlerts')}
            </TabsTrigger>
            <TabsTrigger value="students">
              <UserCircle className="w-4 h-4 mr-2" />
              {t('myStudents')}
            </TabsTrigger>
          </TabsList>

          <TabsContent value="alerts" className="space-y-4">
            {alerts.length === 0 ? (
              <Card>
                <CardContent className="py-12 text-center">
                  <CheckCircle className="w-12 h-12 text-green-500 mx-auto mb-3" />
                  <p className="text-muted-foreground">{t('noAlertsAtThisTime')}</p>
                </CardContent>
              </Card>
            ) : (
              <div className="space-y-3">
                {alerts.map((alert) => (
                  <Card
                    key={alert.id}
                    className={`${
                      alert.status === 'active' ? 'border-red-500 border-2' : ''
                    }`}
                  >
                    <CardHeader>
                      <div className="flex items-start justify-between">
                        <div className="space-y-1">
                          <CardTitle className="flex items-center gap-2">
                            <AlertTriangle
                              className={`w-5 h-5 ${
                                alert.status === 'active' ? 'text-red-500' : 'text-gray-400'
                              }`}
                            />
                            {alert.studentName}
                          </CardTitle>
                          <CardDescription className="flex items-center gap-2">
                            <Clock className="w-3 h-3" />
                            {formatTime(alert.timestamp)}
                          </CardDescription>
                        </div>
                        {getStatusBadge(alert.status)}
                      </div>
                    </CardHeader>
                    <CardContent className="space-y-4">
                      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                        <div>
                          <p className="text-sm font-medium mb-1">{t('medicalCondition')}</p>
                          <Badge variant="destructive">{alert.medicalCondition}</Badge>
                        </div>
                        <div>
                          <p className="text-sm font-medium mb-1 flex items-center gap-1">
                            <MapPin className="w-4 h-4" />
                            {t('location')}
                          </p>
                          <p className="text-sm">{alert.location.description}</p>
                          <p className="text-xs text-muted-foreground">
                            {alert.location.latitude.toFixed(4)}, {alert.location.longitude.toFixed(4)}
                          </p>
                        </div>
                      </div>

                      {alert.status === 'active' && (
                        <div className="flex gap-2">
                          <Button
                            className="flex-1"
                            onClick={() => handleRespondToAlert(alert.id)}
                          >
                            {t('imResponding')}
                          </Button>
                          <Button
                            variant="outline"
                            onClick={() => handleResolveAlert(alert.id)}
                          >
                            {t('markResolved')}
                          </Button>
                        </div>
                      )}

                      {alert.status === 'responded' && (
                        <div className="flex gap-2">
                          <Button
                            variant="outline"
                            className="flex-1"
                            onClick={() => handleResolveAlert(alert.id)}
                          >
                            <CheckCircle className="w-4 h-4 mr-2" />
                            {t('markAsResolved')}
                          </Button>
                        </div>
                      )}
                    </CardContent>
                  </Card>
                ))}
              </div>
            )}
          </TabsContent>

          <TabsContent value="students" className="space-y-3">
            {myStudents.length === 0 ? (
              <Card>
                <CardContent className="py-12 text-center">
                  <p className="text-muted-foreground">{t('noStudentsAssigned')}</p>
                </CardContent>
              </Card>
            ) : (
              myStudents.map((student) => (
                <Card key={student.id}>
                  <CardHeader>
                    <CardTitle>{student.name}</CardTitle>
                    <CardDescription>{student.grade}</CardDescription>
                  </CardHeader>
                  <CardContent className="space-y-3">
                    <div>
                      <p className="text-sm font-medium mb-1">{t('medicalCondition')}</p>
                      <Badge variant="destructive">{student.medicalCondition}</Badge>
                    </div>
                    <div>
                      <p className="text-sm font-medium mb-1 flex items-center gap-1">
                        <Phone className="w-4 h-4" />
                        {t('emergencyContact')}
                      </p>
                      <p className="text-sm">{student.emergencyContact}</p>
                    </div>
                  </CardContent>
                </Card>
              ))
            )}
          </TabsContent>
        </Tabs>
      </div>
    </div>
  );
}
